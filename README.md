# cvo+Trident+EKS
## やりたいこと
- オンプレのAFF8040（ONTAP9.7）+Trident+K8S環境に作成したWordpress環境をSnapMirrorでAWSへ転送し、CVO+Trident+EKS環境に同期する
- CVO+Tridentの連携はCloudManagerのK8S連携機能を使う。極力tridentctlは使わない。
	- EKS+Trident+CVOだと、tridentctlをどこにインストールするのか不明
	- 今後ハンズオントレーニング化を見据え、極力シンプルな環境にしたいため

## 質問したい点/ハマっているポイント（概要）
- オンプレ側のNFSにMysql用PVを作成し、SnapMirrorで転送。CVO側で転送されたVolumeをマウントしようとすると、エラーが発生する。解決する手段はないか？
- CVO側でiSCSIでPod利用するとエラーが発生。解決する手段はないか？
- tridentctl利用が必須とすると、今回の環境ではどこにインストールすべきか？

## 環境
- AWS側
  - EKS:1.18
  - Trident：20.07.1
  - ONTAP：9.9.1RC1
  - Windows端末からGitBashでClusterを操作
  - tridentctl未インストール
- オンプレ側
 	- K8S：v1.20.6
	- Trident：21.01.2
	- ONTAP：9.7P9
  - tridentctlインストール済み

		

## 出来ていること
- NGINXのindex.htmlなどのステートレスなコンテナのデータをSnapMirrorでCVO+Trident+EKS環境へ移行する
1. オンプレK8Sに作ったPod：Nginxのindex.htmlをTridentでNetApp AFF8040に配置
2. AFF8040からCVOへSnapMirror
3. AWS側でCVO＋Trident＋EKSでpod:Nginxを立てて、オンプレと同じIndex.htmlを表示させる
4. tridentctlによるVolume Importは実施していない。
   1. tridentctl未インストールのため
   2. volume imoportしなくてもできるはず！と思っていて実際にできた。
## 質問したい点①
以下手順でオンプレからSnapMirrorしたPVを利用してCVO側でMysqlを立ち上げようとしているがうまくいかない。解決策はないか？

1.オンプレ側のMySQL PodがNFSでマウントしているPVをSnapMirrorでAWS上のCloud Volumes ONTAP（CVO）へ転送

2.CVO側のMySQL Podで転送済みのVolumeをマウントするとPermission Errorが出てマウントできず、Podも作成できない

    $ kubectl logs wordpress-mysql-855db86bd6-6m8xk
    2021-05-05 13:25:56+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 5.6.51-1debian9 started.
    find: '/var/lib/mysql/mysql': Permission denied
    find: '/var/lib/mysql/wordpress': Permission denied
    find: '/var/lib/mysql/performance_schema': Permission denied
    find: '/var/lib/mysql/.snapshot/snapmirror.5c4172a4-ab53-11eb-a2de-a5a9312387ff_2149920682.2021-05-03_093656/mysql': Permission denied
    find: '/var/lib/mysql/.snapshot/snapmirror.5c4172a4-ab53-11eb-a2de-a5a9312387ff_2149920682.2021-05-03_093656/wordpress': Permission denied
    find: '/var/lib/mysql/.snapshot/snapmirror.5c4172a4-ab53-11eb-a2de-a5a9312387ff_2149920682.2021-05-03_093656/performance_schema': Permission denied
    chown: changing ownership of '/var/lib/mysql/.snapshot': Read-only file system

3. Permission Error解決のためオンプレ側Pod内で以下を実行

    ```root@wordpress-mysql-85977cf888-7b789:/var/lib/mysql# chmod 777 *```

強引にすべてのファイルのPermission変えてしまう。
※これはやらなくてもオンプレ側のPV,PVCをReadWriteManyにすればいけるかも？

4. それでも以下のエラーが出て、AWS側でPod作成できない

       $ kubectl logs wordpress-mysql-855db86bd6-p8hwt
       2021-05-29 14:59:12+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 5.6.51-1debian9 started.
       chown: changing ownership of '/var/lib/mysql/.snapshot': Read-only file system

5. しかし'/var/lib/mysql/.snapshot'なんてファイルもディレクトリもPodからはみえないし、CVOのVolumeをマウントしても見えない。SnapMirror用のSnapshotだと思うが、NetAppで-v3-hide-snapshotをenableにしてもどうしても'/var/lib/mysql/.snapshot'がみえてしまう。

6. 色々調べたが、どうもNFS＋Mysql環境だと起きる仕様なのかバグ？

https://jira.percona.com/browse/K8SPXC-286

>NAS-based storages for K8S could create own directories in volumes. For example with default settings NetApp devices adding .snapshot directory.
>This directory prevents SST:
>2020-05-20T10:56:56.288075Z WSREP_SST: [INFO] Proceeding with SST......... rm: cannot remove '/var/lib/mysql/.snapshot': Read-only file system
>xbstream: Can't create/write to file './.snapshot/db.opt' (Errcode: 30 - Read-only file system)
>xbstream: failed to create file.

https://bugs.mysql.com/bug.php?id=53797
>Description:
>We use a NetApp Filer as NFS storage. We mount a NetApp volume as our data directory. NetApp per default has a .snapshot directory for snapshots.

https://github.com/docker-library/mysql/issues/186

https://stackoverflow.com/questions/49945091/latest-mysql-not-coming-up-in-kubernetes/49952285
>This specific error is because --ignore-db-dir is not specified correctly. = should not be used.
>From your logs, the following can be seen
>	[Server] unknown variable 'ignore-db-dir=lost+found'
>Since this fails, the ignore database directory option is not handled correctly.
>Ideally, the additional options that I mentioned below should also be part of the yaml file. Parameters like root password should be set.
>args:
>- "--ignore-db-dir"
>- "lost+found"
>env:

7. --ignore-db-dirとかlost+foundを試したが、解決しない。

## できていないこと/質問したいこと②
- CVO側でiscsi利用したPVをマウントできない
- そこでいったんNFS＋MySQLは諦めることにして、TridentでiSCSI＋MySQLに切り替えることに
- こちらはSnapMirror以前に、CVO側でiscsiを利用できていない
- PV,PVCは問題なくデプロイされるが、Podがあがってこない。
		
	  $ kubectl describe pod wordpress-mysql-xxxx
	  Normal   SuccessfulAttachVolume  11m                  attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-5774e7e5-6062-492f-9fad-cd39e5603f4a"  Warning  FailedMount             7m1s                 
      kubelet                  Unable to attach or mount volumes: unmounted volumes=[mysql-persistent-storage], unattached volumes=[default-token-xr44s mysql-persistent-storage]: timed out waiting for the condition  Warning  FailedMount             45s (x13 over 11m)   
      kubelet                  MountVolume.MountDevice failed for volume "pvc-5774e7e5-6062-492f-9fad-cd39e5603f4a" : rpc error: code = Internal desc = LUN trident_pvc_5774e7e5_6062_492f_9fad_cd39e5603f4a, device sda already formatted with other filesystem: gpt  Warning  FailedMount             16s (x4 over 9m16s)  
      kubelet                  Unable to attach or mount volumes: unmounted volumes=[mysql-persistent-storage], unattached volumes=[mysql-persistent-storage default-token-xr44s]: timed out waiting for the condition
でVolumeのアタッチまでは成功するけど、マウントに失敗する。
			
ファイルシステムが怪しそうなので、StorageClassを確認してみると、

			$ kubectl describe sc vsaworkingenvironment-gfjmakam-single-san
			Name:                  vsaworkingenvironment-gfjmakam-single-san
      IsDefaultClass:        Yes
      Annotations:           storageclass.kubernetes.io/
      is-default-class=true
      Provisioner: csi.trident.netapp.ioParameters:            
      backendType=ontap-san,provisioningType=thin,storagePools=VsaWorkingEnvironment-gfJMAKAM-san_single:.+
      AllowVolumeExpansion:  <unset>
      MountOptions:          <none>
      ReclaimPolicy:         Delete
      VolumeBindingMode:     Immediate
      Events:                <none>
となっていてfsTypeが指定されていない。

そこで、新規にStoragaClassをfsType: ext4指定して作ってみました。PV自体は

    Source:
			    Type:              CSI (a Container Storage Interface (CSI) volume source)
			    Driver:            csi.trident.netapp.io
			    FSType:            ext4
			    VolumeHandle:      pvc-80b1135a-d6df-4808-960a-74585e195e97
			    ReadOnly:          false
			    VolumeAttributes:      backendUUID=174654b7-a2b4-4302-ae2b-775ab0d851d0
			                           internalName=trident_pvc_80b1135a_d6df_4808_960a_74585e195e97
			                           name=pvc-80b1135a-d6df-4808-960a-74585e195e97
			                           protocol=block
			                           storage.kubernetes.io/csiProvisionerIdentity=1624718512255-8081-csi.trident.netapp.io
			Events:                <none>
で問題なさそうですが、PODはやはり

			Warning  FailedMount             7s (x8 over 71s)   
      kubelet                  MountVolume.MountDevice failed for volume "pvc-80b1135a-d6df-4808-960a-74585e195e97" : 
      rpc error: code = Internal desc = LUN trident_pvc_80b1135a_d6df_4808_960a_74585e195e97, device sda already formatted with other filesystem: gpt
とでてマウントに失敗。
			
CloudManagerConnectorがデフォルトで作成するStorageClassでiSCSIを利用する簡単な方法はないか？が聞きたいこと。

## できていないこと/質問したいこと③
- EKS環境ではtridentctlはどこにインストールして使うべきか？
  - ネイティブなK8S環境だとMasterNodeにTrindentインストールするが、EKSだとMasterNodeにはインストールできない
  - CloudManagerから連携されるとTrindentctlインストールしないままステップが進む
  - 管理端末はWindows+GitBash環境なので管理端末にtridentctlはインストールできない
- WokerNodeにインストールすればいいのか？
- EKSやWokerNodeを操作できるLinuxサーバーを用意する必要がある？
