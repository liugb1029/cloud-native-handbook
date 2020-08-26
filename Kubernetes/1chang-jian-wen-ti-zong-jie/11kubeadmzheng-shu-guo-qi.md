\# kubeadm1.16修改证书过期时间

  


一、安装golang, 这里使用v1.12.12

  


二、查看证书过期时间

  


kubeadm alpha certs check-expiration​

  


显示

  


​CERTIFICATE EXPIRES RESIDUAL TIME EXTERNALLY MANAGED

  


admin.conf Oct 13, 2020 08:00 UTC 357d no

  


还剩357天，默认为一年

  


三、查看当前使用版本

  


kubeadm config images list

  


W1022 09:19:08.524437 ......... client version: v1.16.1

  


四、根据上步版本下载源码

  


git clone --branch v1.16.1

\[

https://github.com/kubernetes/kubernetes.git​

\]

\(

https://github.com/kubernetes/kubernetes.git​

\)

  


五、进入源码目录，查看相关的文件

  


cat cmd/kubeadm/app/util/pkiutil/pki

\\_

helpers.go​

  


找到代码NotAfter: time.Now

\\(\\)

.Add

\\(

kubeadmconstants.CertificateValidity

\\)

.UTC

\\(\\)

,​

  


根据import 找到相关文件并修改

  


vim cmd/kubeadm/app/constants/constants.go​

  


将CertificateValidity = time.Hour

\\*

24

\\*

365 改为 time.Hour

\\*

24

\\*

365

\\*

3

  


六、编译

  


make WHAT=cmd/kubeadm GOFLAGS=-v

  


生成的二进制在

\\_

output/bin/ 目录下

  


七、替换

  


先备份

  


cp /usr/bin/kubeadm /usr/bin/kubeadm.backup​

  


再替换

  


cp

\\_

output/bin/kubeadm /usr/bin/kubeadm​

  


八、证书更新

  


1、仍是先备份

  


cp -r /etc/kubernetes/pki /etc/kubernetes/pki.backup​

  


2、开始生成

  


cd /etc/kubernetes/pki

  


kubeadm alpha certs renew all

  


注，如果你生成集群时自己指定的conf, 则需要使用kubeadm alpha certs renew all --config=/yourpath/kubeadm/kubeadm-config.yaml

  


会显示

  


!\[\]

\(

/assets/kubeadm证书-1.png

\)

  


​九、验证更新结果

  


​ kubeadm alpha certs check-expiration

  


!\[\]

\(

/assets/kubeadm证书-2.png

\)

  


证书有效期已是3年，修改成功。

  


  


