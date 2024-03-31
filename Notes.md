gcp-note
===

## 基本檢查

**印出authentication**  
```
gcloud auth list
```  

**印出當前project ID**  
```
gcloud config list project
```

**查看configuration**  
```
gcloud config list
```   

**查看所有**  
```
gcloud config list --all
```   

**查看available instances**  
```
gcloud compute instances list
```   

**查看firewall rules**
```
gcloud compute firewall-rules list
```   

## 前置設定  
**更改local default region**  
```
gcloud config set compute/region [region]
```   

**更改local default zone**  
```
gcloud config set compute/zone [zone]
```  

**把X存成變數，這裡以region值為例**  
```
export region = "[region]"
```   

**也可改成**  
```
export region=$(gcloud config get-value compute/region)
```  

**把zone存成變數**  
```
export zone = "[zone]"
```  

## 開VM instance

```
gcloud compute instances create [VM-name] --machine-type [type] --zone $zone
```  

**用ssh連線到該台instance**  
```
gcloud compute ssh [VM-name] --zone $zone
``` 

**讓該機器安裝nginx**  
```
sudo apt install -y nginx
``` 

**關閉ssh連線**  
```
exit
```   

*The nginx web server is expecting to communicate on tcp:80. To get communication working you need to:*  
- Add a tag to the virtual machine  
- Add a firewall rule for http traffic

**新增標籤，使用逗號分隔來加上更多標籤**   
```
gcloud compute instances add-tags [VM-name] --tags [tag-name]
``` 

**新增firewall-rules讓外部連結能導到VM**  
```
gcloud compute firewall-rules create [firewall-rule-name] --direction=INGRESS --priority=1000 --network=default
--action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=[tag-name]
```
**檢查剛剛的設定，特別filter出allow 80 port**  
```
gcloud compute firewall-rules list --filter=ALLOW:'80'
```  

**測試結果，會出現nginx default page**
```
curl http://$(gcloud compute instances list --filter=name:[VM-name] --format='value(EXTERNAL_IP)')
```  

## 開google kubernetes engine(GKE) cluster  
*A cluster consists of at least one cluster master machine and multiple worker machines called nodes.
Nodes are [Compute Engine virtual machine (VM)](https://cloud.google.com/compute/docs/instances/) instances that run the Kubernetes processes necessary to make them part of the cluster.*  

**開cluster master**  
```
gcloud container clusters create --machine-type=[type] --zone=$zone [cluster-name]
```  

**獲得credentials去跟機器互動**  
```
gcloud container clusters get-credentials [cluster-name]
```   

*GKE uses Kubernetes objects to create and manage your cluster's resources. Kubernetes provides the Deployment object for deploying stateless applications like web servers. 
Service objects define rules and load balancing for accessing your application from the internet.*  

**從[Container Registry](https://cloud.google.com/container-registry/docs)裡的image去部屬app**  
```
kubectl create deployment [application-name] --image=[image-address]
```  

*To create a Kubernetes Service, which is a Kubernetes resource that lets you expose your application to external traffic*  

**expose your kubernetes service**  
```
kubectl expose deployment [application-name] --type=LoadBalancer --port 8080
```  

**檢視服務**  
```
kubectl get service
```  

**輸入網址從瀏覽器檢視**  
```
http://[EXTERNAL-IP]:8080
```   

**刪除cluster**  
```
gcloud container clusters delete [cluster-name]
```   

## network load balancing with [target pool](https://cloud.google.com/load-balancing/docs/target-pools) and [forwarding rules](https://cloud.google.com/load-balancing/docs/forwarding-rule-concepts)
**先創建三個VM分別host不同的page**  
*Forwarding rule與target pool一起使用，以支援load balancing功能*
```
  gcloud compute instances create www1 \
    --zone=Zone \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```  
```
  gcloud compute instances create www2 \
    --zone=Zone \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www2</h3>" | tee /var/www/html/index.html'
```  
```
  gcloud compute instances create www3 \
    --zone=Zone  \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www3</h3>" | tee /var/www/html/index.html'
```

**新增firewall-rules讓外部連結能導到VM**
```gcloud compute firewall-rules create [firewall-rules-name] \
    --target-tags [VM's-tag-name] --allow tcp:80
```

**取得VM對外IP並測試**
```
gcloud compute instances list
curl http://[IP_ADDRESS]
```

**設定load balancing service的邏輯**  

*When you configure the load balancing service, your virtual machine instances receives packets that are destined for the static external IP address you configure.
Instances made with a Compute Engine image are automatically configured to handle this IP address.*  

**創造一個static IP要給load balancer**  
```
gcloud compute addresses create [load-balancer-address-name] \
  --region $region
```

**創造required legacy health check**  
```
gcloud compute http-health-checks create [health-check-name]
```

**創在target-pool，並設定在跟VM同個region，然後加上必要的health-check執行可用性檢查不然不能運行**
```
gcloud compute target-pools create [target-pool-name] \
  --region $region --http-health-check [health-check-name]
```

**把VM instances加入target pool**  
```
gcloud compute target-pools add-instances [target-pool-name] \
    --instances [VM1-name], [VM2-name], [VM3-name]
```

**新增forwarding-rules來接收client requests然後導到target pool裡的VM**  
```
gcloud compute forwarding-rules create www-rule \
    --region  $region \
    --ports 80 \
    --address [load-balancer-address-name] \
    --target-pool [target-pool-name]
```

**把該forwarding rules的外部IP叫出來(就是上面設定過的load-balancer-address-name變數值)**  
```
gcloud compute forwarding-rules describe [forwarding-rules-name] --region $region
```

**access to and show the external IP**  
```
IPADDRESS=$(gcloud compute forwarding-rules describe [forwarding-rules-name] --region $region --format="json" | jq -r .IPAddress)
echo $IPADDRESS
```

**發請求給forwarding rules去測試**  
*replacing IP_ADDRESS with an external IP address from the previous command*  
```
while true; do curl -m1 $IPADDRESS; done
```

## Create an [HTTP load balancer](https://cloud.google.com/load-balancing/docs/application-load-balancer)

*To set up a load balancer with a Compute Engine backend, your VMs need to be in an instance group.
 The managed instance group provides VMs running the backend servers of an external HTTP load balancer.*

**創建template當作開機器的configuration，之後自動開機器時能確保開一樣規格的**  
```
gcloud compute instance-templates create [template-name] \
   --region=$region \
   --network=default \
   --subnet=default \
   --tags=[tag-name] \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```

**創建managed instance group[(MIG)](https://cloud.google.com/compute/docs/instance-groups)**  
```
gcloud compute instance-groups managed create [mig-name] \
   --template=[template-name] --size=[size-of-VM] --zone=$zone
```

**創建firewall-rules**  
```
gcloud compute firewall-rules create [firewall-rules-name] \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=[tag-name] \
  --rules=tcp:80
```

*The ingress rule allows traffic from the Google Cloud health checking systems (130.211.0.0/22 and 35.191.0.0/16).
 Here uses the target tag allow-health-check to identify the VMs*  
 
**創建一個外部IP給load balancer，並打印出來**  

```
gcloud compute addresses create [IP-name] \
  --ip-version=IPV4 \
  --global

gcloud compute addresses describe [IP-name] \
  --format="get(address)" \
  --global
```

**創建[health check](https://cloud.google.com/load-balancing/docs/health-checks)**  

```
gcloud compute health-checks create http [health-check-name] \
  --port 80
```

**創建backend service**  

```
gcloud compute backend-services create [backend-service-name] \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=[health-check-name] \
  --global
```

**把instance group加入到backend service**  

```
gcloud compute backend-services add-backend [backend-service-name] \
  --instance-group=[instance-group--name] \
  --instance-group-zone=$zone \
  --global
```

**創建[URL map](https://cloud.google.com/load-balancing/docs/url-map-concepts)把incoming requests導到backend service**  

```
gcloud compute url-maps create [url-map-name] \
    --default-service [backend-service-name]
```

**正式的創建一個proxy作為load balancer把進來的流量按照url map發送**  

```
gcloud compute target-http-proxies create [proxy-name] \
    --url-map [url-map-name]
```

**創建fowarding rules並把相關流量導到load balancer，可以把forwarding rules視為load balancer本身，因為proxy存在該服務的內部，而唯一外部IP也是給了forwarding rules當作入口**  

```
gcloud compute forwarding-rules create [forwarding-rules-name] \
   --address=[IP-name]\
   --global \
   --target-http-proxy=[proxy-name] \
   --ports=80
```

**從瀏覽器測試結果**  

*replacing IP_ADDRESS with the load balancer's IP address.*  

```
http://IP_ADDRESS/
```
