# Pacemaker Test (RHEL 7, VMware)

<aside>
⚠️ 사내 테스트 서버는 vCenter 환경이므로 power fence는 fence_vmware_rest를 사용

베어메탈 환경에서 stonith 구성시 IPMI를 통해 구성해야함, 펜싱 네트워크는 일반적으로 iLO 네트워크망을 통해서 서버 바이오스에 진입하게되고, 재부팅을 할 수 있게됨.

</aside>

<aside>
✅ 모든 노드, Active 노드 유의해서 입력!

</aside>

# Overview

| hostname | OS | IP |
| --- | --- | --- |
| PACEMAKERT01
(Active 노드) | RHEL 7.9 | 172.16.0.50(RIP)
10.0.1.20(HB) |
| PACEMAKERT02
(Standby 노드) | RHEL 7.9 | 172.16.0.51(RIP)
10.0.1.21(HB) |
| PACEMAKERISCSI01
(iSCSI 서버) | RHEL 8.6 | 172.16.0.52(RIP) |
- VIP = 172.16.0.53

# 사전 설정 (모든 노드)

## SELinux 비활성화

`vi /etc/default/grub`

```bash
...
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet **selinux=0**"
GRUB_DISABLE_RECOVERY="true"
```

(Legacy) `grub2-mkconfig -o /boot/grub2/grub.cfg`

(UEFI) `grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg`

## firewall 비활성화

`systemctl disable --now firewalld`

## 호스트 이름 설정

`vi /etc/hosts`

```bash
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.0.1.20      PACEMAKERT01-HB
172.16.0.50    PACEMAKERT01

10.0.1.21      PACEMAKERT02-HB
172.16.0.51    PACEMAKERT02
```

## iSCSI 설정 (모든 노드)

[iSCSI 서버 설정](https://www.notion.so/iSCSI-d37e04ce66854210b1b7140fcd9d90d0?pvs=21)

- iSCSI initiator 설치
    
    `yum install iscsi iscsi-initiator`
    
- iSCSI wwn 설정
    
    `vi /etc/iscsi/initiatorname.iscsi`
    
    ```bash
    # Active node
    InitiatorName=iqn.2023-04.local.pacemaker.pacemakert01:active
    ```
    
    ```bash
    # Standby node
    InitiatorName=iqn.2023-04.local.pacemaker.pacemakert02:standby
    ```
    
- iSCSI 실행
    
    `systemctl enable --now iscsi`
    
    `systemctl enable --now iscsid`
    
- iSCSI 서버 연결 및 디스크 가져오기
    
    `iscsiadm -m discovery -t st -p 172.16.0.52`
    
    ```bash
    [root@PACEMAKERT01 ~]# iscsiadm -m discovery -t st -p 172.16.0.52
    172.16.0.52:3260,1 iqn.2023-04.local.pacemaker.test:path03
    172.16.0.52:3260,1 iqn.2023-04.local.pacemaker.test:path04
    ```
    
- 디스크 연결하기
    
    `iscsiadm -m node -T iqn.2023-04.local.pacemaker.test:path03 -p 172.16.0.52 -l`
    
    `iscsiadm -m node -T iqn.2023-04.local.pacemaker.test:path04 -p 172.16.0.52 -l`
    
    `lsblk`
    
    ```bash
    [root@PACEMAKERT01 ~]# lsblk
    NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    sda                   8:0    0   60G  0 disk
    ├─sda1                8:1    0    1G  0 part  /boot
    └─sda2                8:2    0   59G  0 part
      ├─rhel-root       253:0    0 35.6G  0 lvm   /
      ├─rhel-swap       253:1    0    6G  0 lvm   [SWAP]
      └─rhel-home       253:2    0 17.4G  0 lvm   /home
    sdb                   8:16   0   10G  0 disk
    sdc                   8:32   0   10G  0 disk
    sr0                  11:0    1  4.2G  0 rom
    ```
    
    ⇒ 같은 PV이지만 멀티패스가 활성화 되지 않아 따로 표기되고 있음
    

## Multipath 설정 (모든 노드)

- multipath.conf 파일 복붙
    
    `cp /usr/share/doc/device-mapper-multipath-0.4.9/multipath.conf /etc/multipath.conf`
    
- wwn 확인하기
    
    `lsscsi -i`
    
    ```bash
    [root@PACEMAKERT01 ~]# lsscsi -i
    [0:0:0:0]    disk    VMware   Virtual disk     2.0   /dev/sda   -
    [3:0:0:0]    cd/dvd  NECVMWar VMware SATA CD00 1.00  /dev/sr0   -
    [33:0:0:0]   disk    LIO-ORG  lun1             4.0   /dev/sdc   **36001405358b3e557d1b4615b9156ae6e**
    [34:0:0:0]   disk    LIO-ORG  lun1             4.0   /dev/sdb   **36001405358b3e557d1b4615b9156ae6e**
    ```
    
    `/dev/sdb` `/dev/sdc` wwn이 같음
    
- `/etc/multipath.conf` 설정
    
    ```bash
    ...
    blacklist {
            devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
            devnode "^vd[a-z]"
    				device {
    								vendor "VMWare"
    								product "Virtual disk"
    				}
    }
    
    multipaths {
            multipath {
                    wwid                    36001405358b3e557d1b4615b9156ae6e
                    alias                   mpath1
            }
    }
    ```
    
    (맨 아래에 추가 후 저장)
    
- `systemctl enable --now multipathd`
- `lsblk`
    
    ```bash
    [root@PACEMAKERT01 ~]# lsblk
    NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    sda                   8:0    0   60G  0 disk
    ├─sda1                8:1    0    1G  0 part  /boot
    └─sda2                8:2    0   59G  0 part
      ├─rhel-root       253:0    0 35.6G  0 lvm   /
      ├─rhel-swap       253:1    0    6G  0 lvm   [SWAP]
      └─rhel-home       253:2    0 17.4G  0 lvm   /home
    sdb                   8:16   0   10G  0 disk
    └─mpath1            253:3    0   10G  0 mpath
    sdc                   8:32   0   10G  0 disk
    └─mpath1            253:3    0   10G  0 mpath
    sr0                  11:0    1  4.2G  0 rom
    ```
    
    mpath1로 같은 디스크가 하나로 묶여있음
    

## postgresql14-server 설치 (모든 노드)

- repository 다운로드
    
    `yum install -y [https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm](https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm)`
    
- postgresql14 설치
    
    `yum install -y postgresql14 postgresql14-server`
    
- postgresql database 초기화 (데몬 키려면 필요)
    
    `/usr/pgsql-14/bin/postgresql-14-setup initdb`
    
- postgresql 활성화
    
    `systemctl enable --now postgresql-14`
    

사전 설정이 모두 완료되면 모든 노드를 reboot하고 문제되는 부분이 없는지 확인해야한다.

# Pacemaker 설치, 클러스터 세팅

- 해당 문서는 Disconnected 환경을 기준으로 작성

## 패키지 설치 (모든 노드)

- Connected 환경:  `rhel-ha-for-rhel-7-server-rpms` 레포지토리를 활성화 후 설치 진행
    
    `subscription-manager repos --enable=rhel-ha-for-rhel-7-server-rpms`
    
    `yum install pcs pacemaker corosync resource-agents fence-agents-all`
    
- Disconnected 환경: Pacemaker 설치에 필요한 의존패키지 챙겨서 로컬 레포지토리 구축 후 설치 진행
    
    `vi /etc/yum.repos.d/ha.repo`
    
    ```bash
    [HA]
    name=ha
    baseurl={mount_path}/addons/HighAvailability
    enabled=1
    gpgcheck=0
    ```
    
    `yum clean all; yum repolist`
    
    `yum install pcs pacemaker corosync resource-agents fence-agents-all`
    

## 클러스터 설치 (모든 노드)

- pcsd 활성화 및 start
    
    `systemctl enable --now pcsd`
    
- 클러스터 유저 생성
    
    `cat /etc/passwd | grep hacluster` ⇒ hacluster 유저 있는지 확인
    
    `echo P@ssw0rd | passwd hacluster --stdin`
    
- 클러스터 노드 인증
    
    `pcs cluster auth -u hacluster -p P@ssw0rd PACEMAKERT01-HB PACEMAKERT02-HB`
    
    ```bash
    [root@PACEMAKERT01 ~]# pcs cluster auth -u hacluster -p P@ssw0rd PACEMAKERT01-HB PACEMAKERT02-HB
    PACEMAKERT01-HB: Authorized
    PACEMAKERT02-HB: Authorized
    ```
    

## 클러스터 Setup, 활성화 및 시작, 세팅 (Active 노드)

- 클러스터 setup
    
    <aside>
    ✅ HB망을 메인으로 설정해야함, 서비스 IP가 메인으로 올 경우 token을 주고 받을 때 HB로 직접가지 않고 서비스 IP로 한번 더 돌기 때문에 문제가 생길 수 있음.
    `~~pcs cluster setup --name HA-test PACEMAKERT01,PACEMAKERT01-HB PACEMAKERT02,PACEMAKERT02-HB~~` (X)
    
    </aside>
    
    <aside>
    ✅ 클러스터 생성 시 token 값을 옵션으로 줄 수 있다. 기본값은 1000(ms)이며, 여기에서는 10000으로 설정
    설정한 토큰값을 기준으로 노드가 토큰을 서로 주고받으며 정상인지 확인한다.
    
    </aside>
    
    `pcs cluster setup --name HA-test PACEMAKERT01-HB,PACEMAKERT01 PACEMAKERT02-HB,PACEMAKERT02 --token 10000`
    
    - 출력 로그
        
        ```bash
        [root@PACEMAKERT02 ~]# pcs cluster setup --name HA-test PACEMAKERT01-HB,PACEMAKERT01 PACEMAKERT02-HB,PACEMAKERT02
        Destroying cluster on nodes: PACEMAKERT01-HB, PACEMAKERT02-HB...
        PACEMAKERT01-HB: Stopping Cluster (pacemaker)...
        PACEMAKERT02-HB: Stopping Cluster (pacemaker)...
        PACEMAKERT01-HB: Successfully destroyed cluster
        PACEMAKERT02-HB: Successfully destroyed cluster
        
        Sending 'pacemaker_remote authkey' to 'PACEMAKERT01-HB', 'PACEMAKERT02-HB'
        PACEMAKERT01-HB: successful distribution of the file 'pacemaker_remote authkey'
        PACEMAKERT02-HB: successful distribution of the file 'pacemaker_remote authkey'
        Sending cluster config files to the nodes...
        PACEMAKERT01-HB: Succeeded
        PACEMAKERT02-HB: Succeeded
        
        Synchronizing pcsd certificates on nodes PACEMAKERT01-HB, PACEMAKERT02-HB...
        PACEMAKERT01-HB: Success
        PACEMAKERT02-HB: Success
        Restarting pcsd on the nodes in order to reload the certificates...
        PACEMAKERT01-HB: Success
        PACEMAKERT02-HB: Success
        [root@PACEMAKERT02 ~]#
        ```
        
    
- corosync 설정 확인
    - `vi /etc/corosync/corosync.conf`
        
        ```bash
        totem {
            version: 2
            cluster_name: HA-test
            secauth: off
            transport: udpu
            rrp_mode: passive
        		token: 10000
        }
        
        nodelist {
            node {
                ring0_addr: PACEMAKERT01-HB
                ring1_addr: PACEMAKERT01
                nodeid: 1
            }
        
            node {
                ring0_addr: PACEMAKERT02-HB
                ring1_addr: PACEMAKERT02
                nodeid: 2
            }
        }
        
        quorum {
            provider: corosync_votequorum
            two_node: 1
        }
        
        logging {
            to_logfile: yes
            logfile: /var/log/cluster/corosync.log
            to_syslog: yes
        }
        ```
        
    
- 클러스터 start
    
    `pcs cluster start --all` `pcs cluster enable --all`
    
    ```bash
    [root@PACEMAKERT01 ~]# pcs cluster start --all
    PACEMAKERT01-HB: Starting Cluster (corosync)...
    PACEMAKERT02-HB: Starting Cluster (corosync)...
    PACEMAKERT01-HB: Starting Cluster (pacemaker)...
    PACEMAKERT02-HB: Starting Cluster (pacemaker)...
    ```
    

## 클러스터 상태 확인 (Active 노드)

- `pcs status`
    
    ```bash
    [root@PACEMAKERT01 ~]# pcs status
    Cluster name: HA-test
    
    **WARNINGS:
    No stonith devices and stonith-enabled is not false**
    
    Stack: corosync
    Current DC: PACEMAKERT02-HB (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
    Last updated: Thu May  4 14:10:45 2023
    Last change: Thu May  4 14:10:42 2023 by hacluster via crmd on PACEMAKERT02-HB
    
    2 nodes configured
    0 resource instances configured
    
    Online: [ PACEMAKERT01-HB PACEMAKERT02-HB ]
    
    No resources
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled
    ```
    
    <aside>
    ✅ 경고 문구(No stonith devices and stonith-enabled is not false)가 출력되는데, 현재 클러스터에 stonith를 설정하지 않아서 출력되는 문제
    
    </aside>
    

## 클러스터 property 세팅 (Active 노드)

- stonith를 세팅할 때까지 경고문구가 뜨지 않도록 property 설정
    
    `pcs property set stonith-enabled=false` 
    
    ⇒ stonith 관련 경고문구가 출력되지 않도록 property 설정, 세팅 후에는 true로 바꿔줘야함
    
- maintence-mode로 바꿔 stonith가 발생하지않도록 설정
    
    `pcs property set maintenance-mode=true`
    
    <aside>
    ✅ maintence-mode는 pacemaker 실행중에 리소스 변경이 일어날 경우 의도치 않은 fencing이 일어날 수 있는데, 이를 방지하기위해서 사용함
    
    </aside>
    

- property 세팅 확인
    
    `pcs property`
    
    ```bash
    [root@PACEMAKERT01 ~]# pcs property
    Cluster Properties:
     cluster-infrastructure: corosync
     cluster-name: HA-test
     dc-version: 1.1.23-1.el7_9.1-9acf116022
     have-watchdog: false
     **maintenance-mode: true
     stonith-enabled: false**
    ```
    

### *resource-stickness 설정

<aside>
✅ 페이스메이커는 노드에 score를 부여하는데, score가 높은 노드가 리소스를 가져갈 권한을 가지게 된다.

여기에는 문제점이 하나 있는데, 예를 들어 A노드와 B노드가 있고, A노드가 score 10점, B노드가 score 5점이라고 가정했을 때 A노드에 문제가 생겨 펜싱되었을 때 리소스는 B에 할당되어야 한다.

하지만 리소스는 재할당 될 때 각 노드의 score를 살펴보는데, A노드의 score가 더 높으므로 A로 다시 재할당 된다. A 서버가 죽은 상태에서 다시 리소스가 재할당 되는 것은 의미없는 행동이 되므로 이를 방지하기위해 resource-stickness를 사용하게 된다.

⇒ 펜싱된 노드로의 리소스 롤백을 방지하기 위해 resource-stickness 속성을 사용함, 기본값은 100으로 준다.

</aside>

- `pcs resource defaults update resource-stickness=200`

### *migration-threshold 설정

<aside>
✅ migration-threshold는 리소스 모니터링이 실패했을 경우, 바로 failover를 진행하지 않고 특정 횟수 만큼 다시 기회를 주는 옵션이다. 순간적인 네트워크 통신 실패로 인해 fencing이 발생하는 것을 방지하기 위해 추가하는 옵션

예를들어 migration-threshold=3 으로 설정해놓은 상태에서 Active노드에 postgresql이 멈췄을 때, 3번정도 재부팅을 시도해보고 안되면 fencing을 날리게된다. 기본값은 3으로 준다.

</aside>

- `pcs resource defaults update migration-threshold=3`

# stonith 설정

## fence_kdump 설정

<aside>
✅ kdump_fence는 노드에 문제가 생겼을 경우 서버를 펜싱하기 전 kdump를 수집할 시간을 벌어주는 역할을 한다.

kdump를 전부 수집하기 전에 fencing을 당했을 경우, kdump는 수집되지 않으므로 서버 문제원인을 알 방법이 없어지므로 이를 방지하기 위해 필요한 에이전트

</aside>

- `/etc/kdump.conf` 수정 (모든 노드)
    
    ```bash
    ...
    #extra_modules gfs2
    #default shell
    #force_rebuild 1
    #force_no_rebuild 1
    #dracut_args --omit-drivers "cfg80211 snd" --add-drivers "ext2 ext3"
    fence_kdump_args -p 7410 -f auto -c 0 -i 10
    fence_kdump_nodes {상대노드}
    ```
    
    맨 아래 fence로 시작하는 두 줄 주석을 해제하고, node1 node2를 호스트네임으로 변경한다.
    
    `systemctl restart kdump`
    
- kdump_fence 생성 (Active 노드)
    
    `pcs stonith create {stonith-name} fence_kdump pcmk_reboot_action="off" pcmk_host_list="{node1} {node2} {...}" timeout={number}s`
    
    ⇒ `pcs stonith create kdump fence_kdump pcmk_reboot_action=off pcmk_host_list="PACEMAKERT01-HB PACEMAKERT02-HB" timeout=150s`
    
    ```bash
    [root@PACEMAKERT01 ~]# pcs stonith
     kdump    (stonith:fence_kdump):  Stopped (unmanaged)
    ```
    
    ⇒ `stonith-enabled=false` 로 되어있어서 Stopped로 출력됨
    
    <aside>
    ✅ `pcmk_reboot_action=off` : fence_kdump에서는 reboot action을 지원하지 않으므로 해당 옵션을 reboot이 아니라 off로 설정해주어야 한다. ([https://access.redhat.com/solutions/875883](https://access.redhat.com/solutions/875883))
    
    `pcmk_host_list` : 펜싱 디바이스가 관리할 호스트 네임을 적어준다. `vi /etc/hosts`에 등록되어있어야 한다.
    
    `timeout` : failed된 노드로 부터 dump 수집 완료 메시지를 받을 때 까지 기다리는 시간으로, 수집해야할 kdump 용량이 많으면 그 만큼 시간도 오래걸리기 때문에 적절한 값을 부여해야한다.
    
    </aside>
    

## fence_vmware_rest 설정 (Active 노드)

<aside>
✅ fence_vmware_rest : vSphere 6.5 이상 버전에서 사용 (fence_vmware_soap 사용시 CPU 스파이크 문제 발생)

fence_vmware_soap: vSphere 6.4 이하 버전에서 사용

펜싱을 날리면 서버를 재부팅해야되는데, 서버를 재부팅하기 위해 필요한 에이전트이다. vCenter의 경우 vCenter 매니저쪽으로 요청이 날라가면 지정한 이름의 VM을 다시 재부팅하는 방식

</aside>

- fence_vmware_rest 생성
    
    `pcs stonith describe fence_vmware_rest` ⇒ 사용할 옵션 체크
    
    `pcs stonith create fence_vmware-01 fence_vmware_rest ipaddr=172.16.0.139 login=administrator@vsphere.local passwd=Wlxlvmffjtm\!1324 ssl=1 ssl_insecure=1 port=RHEL_7.9_Pacemaker_1 pcmk_host_list=PACEMAKERT01-HB **delay=5**`
    
    `pcs stonith create fence_vmware-02 fence_vmware_rest ipaddr=172.16.0.139 login=administrator@vsphere.local passwd=Wlxlvmffjtm\!1324 ssl=1 ssl_insecure=1 port=RHEL_7.9_Pacemaker_2 pcmk_host_list=PACEMAKERT02-HB`
    
    ```bash
    [root@PACEMAKERT01 ~]# pcs stonith
     kdump_fence    (stonith:fence_kdump):  Stopped (unmanaged)
     fence_vmware-01        (stonith:fence_vmware_rest):    Stopped (unmanaged)
     fence_vmware-02        (stonith:fence_vmware_rest):    Stopped (unmanaged)
    ```
    
    <aside>
    ✅ `fence_vmware-01`에만 `delay=5` 옵션을 줬는데, 두 노드가 동시에 shoot을 하지 않게 하기 위해 한 쪽 노드에만 delay를 설정한다.
    
    다른 fence network에서는 `delay` 옵션 입력시 fence 작동되지 않는 이슈가 있음, `pcmk_delay_base`를 대신 사용해야함
    
    </aside>
    
    <aside>
    ✅ stonith를 하나가 아니라 두 개를 설정하는 이유는 두 노드가 서로를 동시에 쏘는 split brain이 발생할 수 있기 때문에 각 노드별로 하나씩 설정해준다.
    
    </aside>
    

# resource 설정 (Active 노드)

<aside>
✅ 현재 클러스터에서 사용할 수 있는 resource는 `pcs resource list` 를 입력하여 확인할 수 있다.

postgresql-14를 설치했을 경우 `systemd:postgresql-14` 가 리스트에 있으며 (14버전 기준) 해당 데몬을 리소스로 등록해서 사용한다.

</aside>

<aside>
❓ resource list에 `ocf:heartbeat:pgsql`가 있는데, 이 리소스는 postgresql을 failover 용도로 사용하는게 아니라 active:active 상태에서 **트래픽을 분산시키는 용도로 사용**되는 리소스이다.

또한 리소스를 사용하기 위해 입력해야할 옵션이 너무 많고 DBA로부터 받아야할 정보가 많아 OS 엔지니어가 단독으로 처리하기에는 어려움이 있다. 

</aside>

## pgvip 리소스 생성

- active에서 standby로 failover되더라도 클라이언트는 같은 아이피로 접근하기 때문에 vip를 설정해주어야한다.
- 아래 명령어 입력하여 리소스 생성
    
    `pcs resource create postgresql-vip ocf:heartbeat:IPaddr2 ip=172.16.0.53 cidr_netmask=24`
    

## postgresql-14 리소스 생성

<aside>
💡 리소스를 생성할 때 어떤 옵션을 줘야할지 모르겠다면 `pcs resource describe {resource-name}` 명령어를 입력하여 확인할 수 있다. 대부분 정해져있는 기본값을 사용한다.

고객사에서 특정 리소스의 기본값을 문의하는 경우도 있는데, 위 명령어를 입력하여 고객사에 전달하면된다. stonith도 마찬가지

</aside>

- 리소스 생성
    
    `pcs resource create postgresql-service systemd:postgresql-14` 
    

## HA-LVM: LVM 리소스 생성(RHEL 7 기준)

<aside>
✅ GFS를 사용할 때(여기에서는 iSCSI), 기본적으로 active 노드에 블록 스토리지를 연결하도록 설정한다.

만약 active 노드에 문제가 발생하여 failover가 발생할 경우 active 노드에 연결된 블록 스토리지 연결은 해제되고 standby 노드로 스토리지가 연결되는데, 이 역할을 하는 리소스가 LVM이다.

LVM는 pv, vg, lv 연결까지 수행한다. 이후에 특정 디렉토리에 마운트 시켜주는 리소스는 아래에 생성할 `ocf:heartbeat:Filesystem` 리소스이다.

</aside>

<aside>
⚠️ HA-LVM 생성은 RHEL 7과 RHEL 8 버전 적용하는 방법이 다르므로 주의해야한다. (RHEL 8 은 LVM-activate 리소스를 사용함)

</aside>

### 사전 설정 (모든 노드)

- VG와 LV모두 미리 생성되어있어야한다.
- locking_type 확인
    
    `vi /etc/lvm/lvm.conf` 
    
    ```bash
    ...
    locking_type=1
    ...
    ```
    
    ⇒ `locking_type` default 값은 1 이므로 따로 수정할 필요 없이 확인만 해주면 됨
    
- LVM volume_list 설정
    
    `lsblk` ⇒ 현재 로컬 스토리지의 vg가 rhel로 잡혀있음
    
    ```bash
    [root@PACEMAKERT01 ~]# lsblk
    NAME                MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    sda                   8:0    0   60G  0 disk
    ├─sda1                8:1    0    1G  0 part  /boot
    └─sda2                8:2    0   59G  0 part
      **├─rhel-root       253:0    0 35.6G  0 lvm   /
      ├─rhel-swap       253:1    0    6G  0 lvm   [SWAP]
      └─rhel-home       253:2    0 17.4G  0 lvm   /home**
    ...
    ```
    
    `vi /etc/lvm/lvm.conf`
    
    ```bash
    ...
    volume_list = [ "rhel" ]
    ...
    ```
    
    <aside>
    ⚠️ HA-LVM을 구성하면 모든 VG가 활성화되지 않게 되고, 페이스메이커의 리소스 에이전트롤 통해 VG가 활성화된다.
    
    따라서 RHEL을 기본으로 설치했을경우 rhel VG 및에 lv가 할당되어있어 문제가 발생할 수 있다. `/etc/lvm/lvm.conf` 파일에 volume_list를 설정해줘서 로컬 VG는 항상 활성화 될 수 있도록(페이스메이커가 아니라 로컬에서 관리하도록) 설정해주어야 한다.
    
    </aside>
    
- VG, LV 태그 추가 (선택)
    
    `vgchange data_vg --addtag pacemaker --config 'activation{volume_list=["@pacemaker"]}’`
    
    `lvchange data_vg/data_lv --addtag pacemaker --config 'activation{volume_list=["@pacemaker"]}'`
    
    ⇒ 태그를 굳이 추가하지 않아도 자동으로 redhat이라는 태그를 pacemaker에서 추가함, 다른 노드에서 VG, LV에 접근하지 못하도록하는 역할
    

설정 후 재부팅 진행

### 리소스 생성

`pcs resource create ha-lvm LVM volgrpname=data_vg exclusive=true`

## Filesystem 리소스 추가 (Active 노드)

<aside>
⚠️ LVM 리소스로 연결된 블록 스토리지의 마운트 포인트를 설정하는 리소스

</aside>

- 리소스 생성
    
    `pcs create resource {resource-name} Filesystem device=/dev/{VG-name}/{LV-name} directory={dir} fstype={type}`
    
    ⇒ `pcs create resource lvm_filesystem Filesystem device=/dev/data_vg/data_lv directory=/data fstype=xfs`
    

# stonith, resource 우선순위 설정 (Active 노드)

<aside>
⚠️ stonith 레벨 설정은 펜싱이 일어났을 때 어떤 stonith를 먼저 수행할지 우선순위를 지정하는 기능, 
현재 구성한 클러스터에는 kdump와 power fence 두 개가 구성되어있고, kdump 수집이 끝난 다음 서버를 죽여야하기 때문에 kdump를 우선순위로 놓아야한다.

</aside>

## stonith 레벨 설정

- kdump가 먼저 수집되고 난 다음에 power fencing을 진행해야하기 때문에 kdump를 우선순위로 놓아야함
    
    `pcs stonith level add 1 PACEMAKERT01-HB kdump`
    
    `pcs stonith level add 1 PACEMAKERT02-HB kdump`
    
    `pcs stonith level add 2 PACEMAKERT01-HB fence_vmware-01`
    
    `pcs stonith level add 2 PACEMAKERT02-HB fence_vmware-02`
    

## 리소스 우선순위 설정: Group

<aside>
⚠️ 보통 여러 리소스가 수행될 때 우선순위를 설정하는데, 리소스 그룹에 추가한 순서대로 리소스가 수행되므로 거의 필수적으로 그룹핑을 해줘야한다.

</aside>

- 그룹 추가
    
    `pcs resource group add {group-name} {resource_1} {resource_2} ... {resource_n}`
    
    ⇒ `pcs resource group add postgresql postgresql-vip postgresql-service ha-lvm lvm_filesystem`
    
    `pcs status`
    
    ```bash
    [root@PACEMAKERT01 ~]# pcs status
    Cluster name: HA-test
    Stack: corosync
    Current DC: PACEMAKERT01-HB (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
    Last updated: Fri May  5 17:11:27 2023
    Last change: Fri May  5 17:09:14 2023 by root via cibadmin on PACEMAKERT01-HB
    
    2 nodes configured
    7 resource instances configured
    
    Online: [ PACEMAKERT01-HB PACEMAKERT02-HB ]
    
    Full list of resources:
    
     kdump  (stonith:fence_kdump):  Started PACEMAKERT01-HB
     fence_vmware-01        (stonith:fence_vmware_rest):    Stopped (unmanaged)
     fence_vmware-02        (stonith:fence_vmware_rest):    Stopped (unmanaged)
     **Resource Group: postgresql**
         **postgresql-vip     (ocf::heartbeat:IPaddr2):       Stopped (unmanaged)
         postgresql-service (service:postgresql-14):        Stopped (unmanaged)
         ha-lvm     (ocf::heartbeat:LVM):   Stopped (unmanaged)
         lvm_filesystem     (ocf::heartbeat:Filesystem):    Stopped (unmanaged)**
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled
    ```
    

# Property 재설정

모든 세팅이 완료되면 maintenance-mode와 stonith-enabled를 다시 설정해준다.

- `pcs property set maintenance-mode=false`
- `pcs property set stonith-enabled=true`

# Constraint 설정 (Active 노드)

## 리소스 그룹 우선순위 설정

<aside>
⚠️ 모든 노드가 정상일 때 리소스를 어디에 할당할지를 정하는 옵션이다.
Pacemaker에는 각 노드마다 Score를 부여할 수 있는데, 점수가 높을수록 우선순위가 높은 노드라고할 수 있다.

명령어는 `pcs constraint location {resource-group-name} prefers {node}={score}` 방식으로 사용한다.

</aside>

- `pcs constraint location postgresql prefers PACEMAKERT01-HB=100`

## self fencing 차단 설정

<aside>
✅ 현재 power fencing을 두 개 세팅한 상태이다.

1. PACEMAKERT02-HB → PACEMAKERT01-HB (fence_vmware-01)
2. PACEMAKERT01-HB → PACEMAKERT01-HB (fence_vmware-02)

펜싱은 무조건 상대노드가 나를 펜싱하도록 해야한다. 간혹 설정이 잘못되어 자기자신을 펜싱하는 경우가 있는데 이를 self fencing이라고 한다. 이를 방지하기위해, 1번 펜싱에는 PACEMAKERT01-HB가 접근하지 못하도록, 2번 펜싱은 PACEMAKERT02-HB가 접근하지 못하도록해야 self fencing이 발생하지 않는다.

</aside>

- self fencing 차단 설정
    
    `pcs constraint location add {constraint_name} {stonith_name} {target-host} -INFINITY`
    
    `pcs constraint location add vmware_constraint-01 fence_vmware-01 PACEMAKERT01-HB -INFINITY`
    
    `pcs constraint location add vmware_constraint-02 fence_vmware-02 PACEMAKERT02-HB -INFINITY`
    

# 클러스터 테스트

## kdump_fence 테스트

- node2에서 아래 명령어를 입력한다.
    
    `fence_kdump -t 3200 -v -n PACEMAKER01-HB`
    
    ```bash
    [root@PACEMAKERT02 ~]# fence_kdump -t 3200 -v -n PACEMAKERT01-HB
    [debug]: options {
    [debug]:     nodename = PACEMAKERT01-HB
    [debug]:     ipport   = 7410
    [debug]:     family   = 0
    [debug]:     count    = 0
    [debug]:     interval = 10
    [debug]:     timeout  = 3200
    [debug]:     verbose  = 1
    [debug]: }
    [debug]: node {
    [debug]:     name = PACEMAKERT01-HB
    [debug]:     addr = 10.0.1.20
    [debug]:     port = 7410
    [debug]:     info = 0xb3b3f0
    [debug]: }
    [debug]: waiting for message from '10.0.1.20'
    ```
    
- node1에서 아래 명령어를 입력한다.
    
    `/usr/libexec/fence_kdump_send -c 1 -i 1 PACEMAKERT02-HB`
    
    node2에서 아래 테스트가 출력되면 성공
    
    ```bash
    ...
    [debug]: received valid message from '10.0.1.20'
    ```
    

## Power fencing 테스트

- `pcs stonith fence PACEMAKERT01-HB` ⇒ kdump_fence 딜레이가 150s이기 때문에 150초 동안 대기 후 수집 완료가  되면 VM이 재부팅되어야함

      `pcs stonith fence PACEMAKERT02-HB`

- fence 통신 체크
    
    `fence_vmware_rest -a 172.16.0.139 -l administrator@vsphere.local -p Wlxlvmffjtm\!1324 --ssl-insecure -z -o list | egrep "{node1_vmname}|{node2_vmname}"`
    
    → vCenter 통신을 확인하는 명령어, 정상적인 경우에는 현재 vCenter에 있는 서버 리스트가 출력된다.
    

## Resource 테스트

- `pcs resource move postgresql` ⇒ PACEMAKERT01에 있는 노드를 PACEMAKERT02로 옮겨보는 테스트
    
    <aside>
    📌 `**resource-stickness` 를 설정했을 경우** 명령어 입력 시 아래와 같은 문구가 출력된다.
    
    ```bash
    [root@PACEMAKERT10 ~]# pcs resource move postgresql
    Warning: Creating location constraint cli-ban-postgresql-on-PACEMAKERT10-HB with a score of -INFINITY for resource postgresql on node PACEMAKERT10-HB.
    This will prevent postgresql from running on PACEMAKERT10-HB until the constraint is removed. This will be the case even if PACEMAKERT10-HB is the last node in the cluster.
    ```
    
    원래는 `resource-stickness` 값을 200으로 설정했기 때문에 리소스가 다른 노드로 옮겨지면 리소스가 옮겨진 노드의 스코어가 200점 증가하게된다.
    
    `resource move` 명령어를 입력하면 수동으로 리소스를 이동시키기 때문에 페이스메이커에서 원래 리소스가 있었던 노드의 score를 `-INFINITY`로 설정하여 리소스가 다시 이동하지 못하도록 설정한다.
    
    따라서 리소스 이동이 정상적으로 된 것을 확인하면 아래와 같이 constraint id를 제거해주어야 한다.
    
    `pcs constraint show --full`
    
    ```bash
    [root@PACEMAKERT10 ~]# pcs constraint show --full
    Location Constraints:
      Resource: fence_vmware_rest-01
        Disabled on: PACEMAKERT10-HB (score:-INFINITY) (id:postgresql-constraint-01)
      Resource: fence_vmware_rest-02
        Disabled on: PACEMAKERT20-HB (score:-INFINITY) (id:postgresql-constraint-02)
      Resource: postgresql
        Enabled on: PACEMAKERT10-HB (score:100) (id:location-postgresql-PACEMAKERT10-HB-100)
        Disabled on: PACEMAKERT10-HB (score:-INFINITY) (role: Started) (id:cli-ban-postgresql-on-PACEMAKERT10-HB)
      Resource: postgresql-vip
        Constraint: location-postgresql-vip
          Rule: boolean-op=or score=-INFINITY  (id:location-postgresql-vip-rule)
            Expression: pingd lt 1  (id:location-postgresql-vip-rule-expr)
            Expression: not_defined pingd  (id:location-postgresql-vip-rule-expr-1)
    Ordering Constraints:
    Colocation Constraints:
    Ticket Constraints:
    ```
    
    `pcs constraint remove cli-ban-postgresql-on-PACEMAKERT10-HB`
    
    </aside>
    

## Filesystem resource 테스트

- Active 노드에서 `umount /data` 입력 후 자동으로 마운트가 되는지 확인