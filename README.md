# 2020 NET 챌린지 시즌7

## 본 설명은 과학기술정보통신부 주최, KOREN연구협력포럼 및 한국정보화진흥원 주관으로 진행된 NET 챌린지 캠프 시즌7의 연구개발 과제입니다.

대학교 : 숭실대학교 전자정보공학부 IT융합전공

팀명 : K.F.C

지도교수 : 김영한

K.F.C 팀 구성원 : 박재욱, 선훈식, 황태관, 조의진

과제 소개 
- 쿠버네티스 환경에서 멀티 클라우드 인프라 구축을 위한 멀티 사이트 클러스터링과 통합 관제 시스템을 구현한 모델을 제시한다. 이를 엣지 클라우드와 접목시켜 KOREN망에서 서울, 광주에 공장과 엣지를 운영하고, 대전에 코어 클라우드를 운영하는 가상의 기업의 스마트 팩토리 사고 방지 및 대응 서비스를 제시한다.

과제 성과
- 다양한 지역에 위치한 클라우드를 오버레이 네트워크로 연결하여 하나의 클러스터 구축
- 여러 지역에 분산되어 있는 인프라를 한 곳에서 관리 및 모니터링 가능
- 클라우드에 문제가 생겨도 쿠버네티스를 통해 해당 클라우드에서 돌아가는 서비스 컨테이너들이 자동적으로 빠르게 다른 곳으로 이동
- 공장 디바이스에서 보내는 data를 통해 빠르게 판단 및 조치하여 사고를 빠르게 방지 및 조치 가능
- 실제 공장의 많은 디바이스들이 한꺼번에 보내는 대용량 data들을 KOREN을 통해 빠르게 수집 및 저장 가능
- 공장 관리자는 한 웹페이지를 통해서 공장 디바이스들의 데이터, 공장의 상태 등을 어디서든 쉽게 확인 가능

## kubernetes cluster 구축 (IPIP,vxlan)

- 지금부터 마스터 노드와 워커노드, Kubernetes, calico의 vxlan mode로 클러스터를 구축하는 법을 알아보겠습니다.

    - 우선 각 노드를 간단하게 설명하자면, 크게 서울, 대전, 광주에 위치한 오픈스택 VM들을 노드로 사용하였습니다.
        
        - 대전 1: kfc-dj (103.22.222.245) => master node
        - 대전 2: kfc-dj2 ( 103.22.222.246) => worker node 1 ( backend)
        - 대전 3: kfc-dj3 ( 103.22.222.247) => worker node 2 ( backend) 
        - 서울 1: master (116.89.189.54) => worker node 3( frontend) <이름만 master고 역할은 worker…이름을 바꾸기 번거로워서 그대로 사용했습니다. 
        - 서울 2: worker-1 (116.89.189.21)=> worker node 4( frontend)
        - 광주 1: ssu-nc-in-gist-1 (116.89.189.210) => worker node 5 (frontend)
    
    - 서울은 116.89.189.0/24 , 대전은 103.22.222.0/24 , 광주는 116.89.189.0/24 로 서로 다른 subnet 대역을 가지고 있습니다. 이제부터 서로 다른 subnet에 있는 노드들로 쿠버네티스 클러스터를 구축해보겠습니다.
    
        1. **Kubernetes 설치**
    
            - Kubernetes 설치를 위해서는 우선 먼저 docker를 설치해야합니다. 왜냐하면 kubernetes는 컨테이너들을 관리해주는 도구이기에 기본적으로 컨테이너가 돌아갈 수 있는 엔진이 설치되어야 하고 이 역할을 docker가 해주기 때문입니다(정확히 말하면 docker engine). 컨테이너들을 master, worker node 모두에서 돌아가기 때문에 클러스터에 포함시킬 모든 node에는 설치해주도록 합니다.

                ![1](https://user-images.githubusercontent.com/47939832/111863979-eafb2680-89a1-11eb-9c72-e58e6f0bdd4d.png)

                ![2](https://user-images.githubusercontent.com/47939832/111863980-ec2c5380-89a1-11eb-989e-3c2a9ad02ea3.png)

            - 이 명령어가 아니라 sudo apt-get install docker 로 진행해도 상관없습니다. 다만, docker.io만다운받는 것은 이건 제 생각이지만 쿠버네티스에서 컨테이너를 돌리기 위해 docker engine만 필요해서 최소한 필요한 부분만 다운받는 것이지 않을까 싶습니다…
            - 다운 받은 후에는 다음 명령어를 통해 docker version을 확인 및 제대로 설치가 되었는지 확인할 수 있습니다. 



            - 다음 명령어를 수행하면 다음과 같은 출력이 나옵니다.
            - 
                ![3](https://user-images.githubusercontent.com/47939832/111863981-ec2c5380-89a1-11eb-97e3-f4f90f6abfaa.png)

                ![4](https://user-images.githubusercontent.com/47939832/111863983-ecc4ea00-89a1-11eb-8a0e-dc3179e50445.png)

            - 혹은 다른 명령어를 통해 더 자세하게 볼 수 있습니다. 다만 쿠버네티스 클러스터를 구축할 때는 크게 필요가 없을 수 있습니다.
                ![5](https://user-images.githubusercontent.com/47939832/111863984-ecc4ea00-89a1-11eb-8a7e-60435eedca97.png)

                ![6](https://user-images.githubusercontent.com/47939832/111863985-ed5d8080-89a1-11eb-9615-1698adbcc3b5.png)

            - Docker 를 설치했으면 docker를 enable시켜 줍니다

                ![7](https://user-images.githubusercontent.com/47939832/111863986-ed5d8080-89a1-11eb-8659-1865d549d342.png)

                이런식으로 아무런 결과가 출력되지 않아야 정상입니다.

            - 이후에 Kubernetes gpg key를 추가해줍니다. 이떄 curl 명령어를 사용하는데 만약 curl 명령어가 없다면 sudo apt install curl 명령어를 먼저 수행해 다운받으시길 바랍니다. 다음 명령어를 수행하신 후 ‘OK’라는 출력이 나오면 정상적으로 작동이 된 것입니다.

                ![8](https://user-images.githubusercontent.com/47939832/111863987-edf61700-89a1-11eb-9de0-615dfdacc66e.png)

                이후 Xenial Kubernetes repository를 추가해줍니다.

                ![9](https://user-images.githubusercontent.com/47939832/111863988-edf61700-89a1-11eb-9e9a-4bd2ae941599.png)

            - 여기까지 진행이 되었으면 이제 kubeadm을 설치합니다. Kubeadm는 kubernetes에서 클러스터를 구축하기 위한 명령어 도구를 지원해줍니다. 기본적으로 kubeadm, kubespray 등.. 여러 명령어 도구가 있지만 여기서는 가장 기본적인 kubeadm을 사용하도록 하겠습니다. 항상 새로운 패키지를 install 할 때는 습관적으로 sudo apt-get update명령어를 수행해 한 번 update해주는 것이 좋습니다.

                ![91](https://user-images.githubusercontent.com/47939832/111863989-ee8ead80-89a1-11eb-955d-c71f562310d8.png)

                앞에서 docker 설치시와 마찬가지로 다음 명령어를 수행해서 kubeadm 설치를 확인할 수 있습니다.

                ![92](https://user-images.githubusercontent.com/47939832/111863991-ee8ead80-89a1-11eb-82bd-1df260d1fd15.png)
                
        2. **Kubernetes deploy**
            
            - Kubernetes 는 swap memory를 사용하는 노드에서는 동작을 하지 않습니다. 따라서 다음 명령어를 통해 각 노드의 swap memory를 비활성화 시켜줍니다. 
                ![93](https://user-images.githubusercontent.com/47939832/111863992-ef274400-89a1-11eb-8e77-a8372604982a.png)
                
                만약 일반 local한 노트북이나 데스크탑 등을 Kubernetes node로 사용한다면 컴퓨터를 부팅할 때 마다 swapoff를 해줘야합니다. 그러나 만약 구글이나 아마존 등의 iaas 서비스를 이용하여 VM에서 수행 중이라면 해당 서버가 계속 켜져 있을 테니 굳이 다시 해줄 필요는 없습니다. 그러나 습관적으로 해줘도 상관없습니다.
                
            - 그럼 이제부터 클러스터의 master 노드 초기화 과정을 살펴보겠습니다. (다음 과정부터는 master 노드에서만 진행)
                
                먼저 클러스터 내의 pod들의 CIDR을 설정해줍니다. 
                
                ![94](https://user-images.githubusercontent.com/47939832/111863993-ef274400-89a1-11eb-8bfc-e6edf7d955f7.png)
                
                CIDR을 설정할 때 기본적으로 calico에서는 192.168.0.0/16을 사용하지만 init 명령어에서 관리자가 원하는 주소 대역으로 사용할 수 있습니다. 다만 이 주소 대역이 실제 사용중인 네트워크 내에서 사용되고 있지 않는 private 대역이어야 합니다. 여기서는 10.244.0.0/16대역을 사용하도록 하겠습니다.
                
                이런 식으로 pod들의 CIDR을 설정해주면 클러스터 내에서 새로운 pod 생성시, 항상 pod들은 10.244.0.0/16 대역의 ip주소를 임의로 가지게 됩니다. 
                
                또한 제일 중요한 것은 위 명령어를 수행하면 출력되는 토큰 결과값이 있는데 그 값을 반드시 다른 곳에 저장해놔야합니다(메모장이든 카카오톡이든 언제든 다시 쓸 수 있는 곳에). 그 토큰 결과값을 사용해야 추후에 worker node들을 클러스터에 추가시킬 수 있기 때문입니다. 결과값은 다음과 같은 예시로 출력이 됩니다. 
                
                kubeadm join 103.22.222.245:6443 --token 06tl4c.oqn35jzecidg0r0m --discovery-token-ca-cert-hash sha256:c40f5fa0aba6ba311efcdb0e8cb637ae0eb8ce27b7a03d47be6d966142f2204c
                
                (반드시 저장!)
                
                이어서 kubectl 활성화를 시키겠습니다. kubectl이란 사용자가 master 노드를 통해서 worker노드들에게 명령을 내릴 수 있도록 도와주는 역할을 합니다. Kubernetes 클러스터 내에 명령을 내릴 시 거의 대부분이 kubectl 명령어를 통해서 사용됩니다. 자세한 내용은 kubectl --help를 사용하면 나옵니다.
                
                ![95](https://user-images.githubusercontent.com/47939832/111863994-efbfda80-89a1-11eb-8b20-c99d7b8497f5.png)
                
                이제 master 노드에서 kubectl을 사용할 수 있도록 해줍니다.
                
                ![96](https://user-images.githubusercontent.com/47939832/111863995-efbfda80-89a1-11eb-84f1-a87ec3d60763.png)
                
                이후에 다음 명령어를 수행합니다.
                
                ![97](https://user-images.githubusercontent.com/47939832/111863996-f0587100-89a1-11eb-9075-7d6bcc37aa61.png)
                
                여기서 3 명령어를 한 번에 보여주지 않고 중간에 chown 명령어를 자른 이유를 설명하자면, 먼저 두번째 명령어까지 수행하면 다음 명령어를 통해서 $HOME/.kube 디렉토리 안의 config 파일을 확인할 수 있습니다.
                
                ![98](https://user-images.githubusercontent.com/47939832/111863998-f0587100-89a1-11eb-84a2-1e15c54acf33.png)
                
                확인해보면
                
                ![99](https://user-images.githubusercontent.com/47939832/111863999-f0f10780-89a1-11eb-8c0d-f6b95f3f03bd.png)
                
                이렇게 복잡한 파일이 나옵니다. 이 config파일이 이제 master node에서 kubectl 을 사용할 수 있게 해주는 핵심이라고 생각하면 됩니다. 만약 이 파일이 삭제된다면 kubectl을 사용할 수 없습니다. 그런데 이제 프로젝트를 진행하다 보면 master node가 아닌 worker node에서도 kubectl을 사용해야하는 경우가 있을 수 있습니다. 기본적으로 worker node에서는 kubectl을 사용할 수 없지만 사용 가능하도록 하는 방법이 있습니다. 바로 master node의 config파일을 복사해서 worker node에도 생성하는 것입니다.
                
                ![991](https://user-images.githubusercontent.com/47939832/111864000-f1899e00-89a1-11eb-9efc-4fe6974af857.png)
                
                Worker 노드에서 다음과 같은 명령어를 수행 후 자체적으로 config 파일을 만들어줍니다. 그 후 안에 master의 config파일을 복사해 넣고 3번째 명령어인
                
                ![992](https://user-images.githubusercontent.com/47939832/111864001-f1899e00-89a1-11eb-9b17-c99e2883a63c.png)
                
                을 수행한다면 worker node에서도 master node처럼 동일한 클러스터 내에서 kubectl 명령어를 사용가능합니다. 

                ![993](https://user-images.githubusercontent.com/47939832/111864002-f2223480-89a1-11eb-834c-0baf5c618e9e.png)
                
                주의할 점은 worker에서는 sudo cp -i "--" 는 수행하지 않고 master의 config 복사 후 자체적으로 $HOME/.kube 디렉토리로 들어가 config 생성 후에 sudo chown $(id~~ 명령어를 수행해야합니다.(이 과정은 필수가 아니라 worker에서도 kubectl 사용을 원할 경우에만 수행하면 됩니다)
                
                이후에 kubectl get nodes 명령어를 수행하면 클러스터에 포함된 node들이 출력되고 현재까지는 master node만 출력이 될 것입니다.
                
                이제 클러스터를 구축했으니 calico CNI를 적용시켜 보겠습니다. 기본적으로 kubernetes는 yaml파일을 적용시킴으로써 사용자가 원하는 상태로 만들 수 있고 yaml파일을 적용시키는 명령어는 kubect apply -f [yaml파일] 입니다.
                
                Project calico 홈페이지에서 제공하는 yaml파일을 적용시켜보겠습니다.
                
                ![994](https://user-images.githubusercontent.com/47939832/111864003-f2223480-89a1-11eb-969a-d2883eb70def.png)
                
                다음 명령어를 수행 후 대기하면 calico가 적용되면서 calico-kube-controllers, calico-node 등등 여러 pod들이 추가되는 것을 확인할 수 있습니다.
                
                ![995](https://user-images.githubusercontent.com/47939832/111864004-f2bacb00-89a1-11eb-8fff-5d02e1a89dce.png)
                
                ![996](https://user-images.githubusercontent.com/47939832/111864005-f2bacb00-89a1-11eb-95b3-16180a7bc8e9.png)
                
                기본적으로 calico-node와 proxy는 클러스터에 node를 추가할 때 마다 하나씩 더 생성됩니다. Master 하나만 있을 떄는 calico-node pod도 하나만 있는 것이 정상입니다. 위의 그림은 worker node를 포함해 node가 총 6개여서 저렇게 많이 출력 된 것입니다.
                
            - 이제부터 worker node를 클러스터에 추가시켜보겠습니다. (이후 과정은 master가 아닌 worker로 사용할 node들에서 진행)
                
                이전에 master에서 sudo kubeadm init –pod-network-cidr~ 명령어를 통해 얻은 token값을 다른 곳에 저장해두셨을 겁니다.
                
                kubeadm join 103.22.222.245:6443 --token 06tl4c.oqn35jzecidg0r0m --discovery-token-ca-cert-hash sha256:c40f5fa0aba6ba311efcdb0e8cb637ae0eb8ce27b7a03d47be6d966142f2204c
                
                이런식으로 나왔을 텐데 이걸 그대로 cli창에 입력해줍니다. (sudo 권한이 필요한 경우 sudo를 입력하고 진행합니다). Kubeadm join이란 join 말 그대로 합류시킨다는 의미로 새로운 worker node를 추가시킬 때 마다 진행하는 명령어입니다. 처음 클러스터를 생성할 때 생성된 token값은 영원하지 않으므로 시간이 지난 후 새로운 worker 를 추가시키고 싶을 때는 token값을 새롭게 생성해야합니다(과정은 뒤에서 설명)
                
                각 추가할 node에 모두 명령어 수행이 완료되면 다음 명령어를 통해 클러스터에 포함된 node를 확인가능합니다.
                
                ![997](https://user-images.githubusercontent.com/47939832/111864006-f3536180-89a1-11eb-99d4-343d57052ae8.png)
                
                더 자세한 내용을 확인하고 싶다면 다음 명령어를 수행합니다
                
                ![998](https://user-images.githubusercontent.com/47939832/111864007-f3536180-89a1-11eb-9da7-5d1668e4a971.png)
                
        3. **vxlan mode로 클러스터를 구축하는 방법**
            
            - 현재까지 kubernetes와 calico를 통해 클러스터를 구축하는 과정을 알아봤습니다. Calico의 default mode는 IPIP mode입니다. IPIP와 vxlan 모두 훌륭한 overlay network이지만 Azure같은 경우 IPIP mode를 지원하지 않는 경우도 있고 IPIP를 사용할 시 openstack 보안그룹에서 따로 설정도 해줘야하는 경우가 있을 수 있습니다. 따라서 IPIP 뿐만 아니라 vxlan mode로도 클러스터를 구축하는 방법을 알아보겠습니다.(참고로 이미 IPIP 모드로 클러스터 구축 시, vxlan mode로 변환하려면 calicoctl이라는 명령어를 사용해야합니다. 우선은 처음부터 vxlan mode로 클러스터를 구축하는 법을 알아보겠습니다)
                
                ![999](https://user-images.githubusercontent.com/47939832/111864008-f3ebf800-89a1-11eb-924f-5b06b990b345.png)
                
                이부분 수행 전까지는 모든 과정이 위에서 설명한 것과 동일합니다. 다만, vxlan mode로 변환하기 위해서는 calico의 configmap file을 수정해야합니다. 따라서 여기서는 바로 kubectl apply 를 사용하는 것이 아니라 우선 wget 명령어를 통해 calico.yaml파일을 다운받도록 합시다.
                
                ![9991](https://user-images.githubusercontent.com/47939832/111864009-f3ebf800-89a1-11eb-9886-3fd1ebce02e5.png)
                
                ![9992](https://user-images.githubusercontent.com/47939832/111864011-f4848e80-89a1-11eb-89a0-ec5e2a303d89.png)
                
                명령어를 수행하면 calico.yaml을 받을 수 있고 vim을 통해서 수정합니다
                
                먼저 해줘야 할 것은 configmap 파일의 bird를 vxlan으로 수정하는 것입니다. Bird는 IPIP모드에서 BGP protocol을 사용하기 위한 인터페이스로 vxlan에서는 사용하지 않습니다. 따라서 bird를 vxlan으로 수정합니다
                
                ![9993](https://user-images.githubusercontent.com/47939832/111864012-f4848e80-89a1-11eb-9b9c-f8a15b5eedfc.png)
                
                두번쨰로 해야할 것은 Daemonset 파일에서 IPIP mode의 값을 Always=> Never로 , VxlanMode 값을 Never=> Always 로 수정합니다.
                
                ![9994](https://user-images.githubusercontent.com/47939832/111864013-f51d2500-89a1-11eb-9fb1-5f59731f99b1.png)
                
                ![9995](https://user-images.githubusercontent.com/47939832/111864014-f51d2500-89a1-11eb-91a8-063b9e3a7038.png)
                
                ![9996](https://user-images.githubusercontent.com/47939832/111864016-f5b5bb80-89a1-11eb-8c59-541aa397bd58.png)
                
                3번째로 Daemonset 파일에서 livenessProbe.exec.command의 bird-live와 readinessProbe.exec.command의 bird-ready를 각각 주석처리해준다. 이유는 앞서 설명했듯이 bird는 IPIP에서 사용하므로 vxlan에서는 필요없기 때문입니다.
                
                ![9997](https://user-images.githubusercontent.com/47939832/111864017-f5b5bb80-89a1-11eb-9290-2908a266f08a.png)
                
                ![9998](https://user-images.githubusercontent.com/47939832/111864018-f64e5200-89a1-11eb-8dfb-52f1d30a72dd.png)
                
                총 3가지의 과정을 거치면 vxlan mode로 클러스터를 구축할 수 있다. 뒤의 과정은 IPIP와 동일하게 수행하면 된다.
                
                이후 route -n 명령어로 라우팅 테이블을 확인해보면
                
                ![9999](https://user-images.githubusercontent.com/47939832/111864019-f64e5200-89a1-11eb-9bd7-5a92853dec8f.png)
                
                다음과 같이 각 node마다 vxlan.calico라는 vxlan interface가 생성된 것을 확인할 수 있다.

## 쿠버네티스 환경 설정 과정 中 Calico 세팅 방법


- 앞서 kubernetes와 calico를 이용해 멀티 사이트 클러스터를 구축했습니다. 이어서 calico에서 지원하는 calicoctl 명령어를 알아보도록 하겠습니다. 본 정리에서는 calicoctl 명령어를 /usr/local/bin 디렉토리에 설치하였습니다.(calico 공식 홈페이지에서 예시로 든 디렉토리)
    
    다음 명령어를 통해 calicoctl binary를 설치합니다.
    
    ![1](https://user-images.githubusercontent.com/47939832/111865377-0b2ee380-89aa-11eb-80b9-da776518d6bc.png)
    
    이후에 calicoctl의 권한을 수정해주기 위해 다음 명령어를 실행합니다.
    
    ![2](https://user-images.githubusercontent.com/47939832/111865383-1255f180-89aa-11eb-86b1-7437ad693bbd.png)
    
    Calicoctl을 설치하기 위해서 docker hub에서 pull 받아옵니다.
    
    ![3](https://user-images.githubusercontent.com/47939832/111865384-12ee8800-89aa-11eb-96cf-bef09148a598.png)
    
    이후 calicoctl을 Kubernetes cluster에 배포합니다.
    
    ![4](https://user-images.githubusercontent.com/47939832/111865385-13871e80-89aa-11eb-885d-278b01ec5d23.png)
    
    ![5](https://user-images.githubusercontent.com/47939832/111865387-13871e80-89aa-11eb-97cc-a8929db1e02e.png)
    
    Calicoctl이 설치된 것을 확인할 수 있습니다.
    
    배포 후에 calicoctl 명령어를 설정해줍니다.
    
    ![6](https://user-images.githubusercontent.com/47939832/111865388-141fb500-89aa-11eb-9d8b-4f940a2e1007.png)
    
    보면 알겠지만 calicoctl은 결국 kubectl을 사용한 긴 명령어를 alias 시킨 것이다. 이후 calicoctl을 간단하게 사용해보면
    
    ![7](https://user-images.githubusercontent.com/47939832/111865390-141fb500-89aa-11eb-8f4b-268859215a82.png)
    
    이처럼 kubectl get nodes 를 했을 때 처럼 클러스터 내의 node들이 출력이 되는 것을 확인할 수 있습니다. 
    
    앞서 진행했던
    
    ![8](https://user-images.githubusercontent.com/47939832/111865391-14b84b80-89aa-11eb-95fc-70847dacd1e2.png)
    
    에서 wget 명령어로 calicoctl.yaml을 다운받아 vim으로 열어보면
    
    ![9](https://user-images.githubusercontent.com/47939832/111865393-14b84b80-89aa-11eb-8901-6bc231f2ef94.png)
    
    Calicoctl 명령어로 확인 및 수정할 수 있는 resource들을 확인할 수 있다. 여기서는 ippool을 확인해보기 위해 다음 명령어를 사용합니다.
    
    ![91](https://user-images.githubusercontent.com/47939832/111865394-1550e200-89aa-11eb-8026-54966548104d.png)
    
    Ippool을 확인해보면 초기에 클러스터 구축시 kubeadm init 명령어로 pod-cidr을 설정할 때 설정했던 10.234.0.0/16을 확인할 수 있다. 이를 자세히 보기 위해 다음 명령어를 사용합니다
    
    ![92](https://user-images.githubusercontent.com/47939832/111865395-1550e200-89aa-11eb-9c5f-633cad2662b6.png)
    
    ![93](https://user-images.githubusercontent.com/47939832/111865397-15e97880-89aa-11eb-909a-50100d029171.png)
    
    그럼 다음과 같은 ippool file을 확인할 수 있고 처음 클러스터 구축할 때 vxlan mode로 설정했기 때문에 IPIP mode는 Never, vxlan mode가 always로 되어있는 걸 알 수 있습니다. 처음 클러스터 구축 할 때 만약 IPIP mode로 구축하면 vxlan mode로 바꿀 수 없으므로 처음부터 wget 명령어로 calico.yaml을 받아와 수정하여 vxlan mode로 바꿨던 것을 기억해봅시다. 그럼 만약 vxlan mode로 클러스터를 구축하고 컨테이너들을 배포했는데 IPIP mode로 전환이 필요하다면 클러스터를 삭제하고 다시 해야할까요?? 그런 번거로움을 여기서 해결할 수 있습니다. ippool에서 IPIP mode를 다시 Always로 수정하고 vxlan mode를 Never로 수정한 후에 calicoctl apply를 해주면 됩니다.
    
    ![94](https://user-images.githubusercontent.com/47939832/111865399-15e97880-89aa-11eb-9cd1-a215b20a5aac.png)
    
    ![95](https://user-images.githubusercontent.com/47939832/111865400-16820f00-89aa-11eb-93e1-aedfa6c7b81d.png)
    

## 쿠버네티스 환경에서의 helm설치 및 helm을 이용해 프로메테우스, 그라파나 배포

- **helm 2.0 설치 방법**
    - 헬름 클라이언트는 깃허브에서 다운로드 가능
        
        > wget https://get.helm.sh/helm-v2.16.6-linux-amd64.tar.gz
        > 
        > tar xvzf helm-v2.16.6-linux-amd64.tar.gz
        > 
        > sudo cplinux-amd64/tiller/usr/local/bin
        > 
        > sudo cplinux-amd64/helm/usr/local/bin
        > 
        > sudo chownroot:docker/usr/local/bin/tiller
        > 
        > sudo chownroot:docker/usr/local/bin/helm
    
    - 헬름이 잘 설치 되어 있는지 확인
        > helm version
        > 
        
        -> 확인을 해보면 클라이언트는 정상적으로 설치되었지만, 'helm server-틸러'는 에러가 나는 것을 알 수 있습니다.
        
    - helm server - tiller 설치하기
        
        ~~~
        apiVersion: v1
        kind: ServiceAccount
        metadata:
            name: tiller
          namespace: kube-system
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: tiller
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: cluster-admin
        subjects:
          - kind: ServiceAccount
                name: tiller
            namespace: kube-system
        ~~~
        
    - 이후 kubectl apply -f tiller.yaml 명령어를 입력하여 클러스터에 적용시켜줍니다.

    - tiller 설치
        
        > helm init--service-account tiller
        
    - tiller 설치 확인
        
        > helm version
        > 
        
        -> 클라이언트와 서버가 정상적으로 동작되는 것을 확인할 수 있습니다.
        
    - helm 저장소 업데이트
        
        > helm repo update
        > 

- **helm chart를 활용하여 프로메테우스, 그라파나, alertmanager 쿠버네티드에 적용하기 ( 이 방법은 helm 2 version으로 진행한 것입니다. )**
    
    - helm chart 설치
        
        helm chart는 쿠버네티스에서 package manager 역할을 합니다. 이때, chart를 통해 yaml 파일을 등록하여 저장하는 저장소의 역할을 합니다. 쿠버네티스만을 이용한다면 컨테이너 배포 및 환경 세팅하는 과정에서 어려움을 느낄 수 있습니다. 이러한 이유로 개발자들은 환경 세팅하는 것에서부터 포기하는 경우가 많습니다. 이러한 문제점을 해결해준 것이 helm입니다.
        
        helm의 repository는 깃허브에 등록되어 있어 누구나 다운로드를 받아 사용하실 수 있습니다.
        
        > git clone https://github.com/helm/charts.git
        > 
        
        -> 위와 같이 깃허브에서 다운로드를 받으면 chart repository를 확인할 수 있으며 다양한 패키지들이 제공됩니다. helm에서는 보통 안정된 버전이 Chart를 활용하는 것을 추천합니다.
        
        ![1](https://user-images.githubusercontent.com/47939832/111865926-55fe2a80-89ad-11eb-8d22-9d79e73f9610.png)
        
    - ../stable -> grafana, kube-state-metrics, prometheus-node-exporter, prometheus-operator 패키지 설치
        
        우선적으로 그라파나와 프로메테우스, alertmanager를 동작시키기 위해서 stable 디렉토리에 존재하는 grafana, kube-state-metrics, prometheus-node-exporter, prometheus-operator 디렉토리들을 복사하여 하나의 디렉토리 파일로 새롭게 생성합니다.
        
        ![2](https://user-images.githubusercontent.com/47939832/111865929-572f5780-89ad-11eb-95aa-1e202c43debd.png)
        
        하나의 chart는 'templates'와 'values'로 구성되어 있습니다. 이때, templates는 쿠버네티스의 deployment, service 등의 쿠버네티스 오브젝트들의 templates를 정희해 놓은 yaml 파일 들입니다. values.yaml 파일은 template의 yaml 파일들에 적용할 변수들을 넣어줄 파일입니다.
        
        chart는 참조해야 하는 라이브러리 파일입니다.
        
        정리하자면 하나의 chart를 등록하려면 템플릿에 정의 된 yaml파일을 만들기 위해 values.yaml 파일을 수정하며 또 다른 chart의 패키징된 파일을 저장해야 합니다.
        
        helm은 client-server 구조를 취하고 있습니다.(helm 3은 클라이언트 서버가 통합되어 있습니다.) 즉, 사용자가 사용하는 CLI(command line interface)가 helm이며 쿠버네티스에서 배포하고 관리하는 서버를 tiller라고 부릅니다. 그래서 우선적으로 helm을 설치하고 tiller를 서버에 배포하여 사용하게 되는 것입니다.
        
    - node에 label을 붙여 helm chart 관리하기 -> kfc-dj2노드에 key=monitoring 노드라벨 붙이기
        
        ![3](https://user-images.githubusercontent.com/47939832/111865930-572f5780-89ad-11eb-8f43-64387b474a81.png)
        
    - kfc-dj2 노드에 key=monitoring 라벨 붙이기
        
        ![4](https://user-images.githubusercontent.com/47939832/111865932-57c7ee00-89ad-11eb-86ce-ff09aee08ef4.png)
        
        각 패키지 라이브러리의 values.yaml 파일로 들어가 nodeSelector 부분을 key : monitoring으로 수정해 줍니다.
        
    - grafana, kube-state-metrics, prometheus-node-exporter에 대하여 패키지화 하여 charts 디렉토리를 생성해 charts디렉토리에 추가
        
        ~~~
        helm package grafana/
        helm package kube-state-metrics/
        helm package prometheus-node-exporter/
        mv *.tgz prometheus-operator/charts/
        ~~~
        
        chars 디렉토리에 grafana, kube-state-metrics, prometheus-node-exporter 패키지 파일들이 존재하는지 확인합니다.
        
        ~~~
        ls -la prometheus-operator/charts/
        ~~~
        
        ![5](https://user-images.githubusercontent.com/47939832/111865933-57c7ee00-89ad-11eb-98a8-795cc51c4039.png)
        
    - grafana와 prometheus alertmanage가 사용하는 pv.yaml 파일 등록
        
        ~~~
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: pv-for-alertmanager
        spec:
          capacity:
           storage: 50Gi
         volumeMode: Filesystem
          accessModes:
            - ReadWriteOnce
          persistentVolumeReclaimPolicy: Retain
          nfs:
            path: /root/nfs/alertmanager
            server: 103.22.222.246
        ---
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: pv-for-grafana
        spec:
          capacity:
            storage: 10Gi
          volumeMode: Filesystem
          accessModes:
            - ReadWriteOnce
          persistentVolumeReclaimPolicy: Retain
          nfs:
            path: /root/nfs/grafana
            server: 103.22.222.246
        ~~~
        
        ![6](https://user-images.githubusercontent.com/47939832/111865934-58608480-89ad-11eb-81ba-9fa111ac3003.png)
        
        -> 이때 server의 IP는 kfc-dj2 즉, key=monitoring 라벨을 붙인 노드의 IP입니다.	
        
        이후, kubectl create -f pv.yaml 커멘드를 입력합니다.
        
    - helm install을 활용한 helm chart 등록
        
        ![7](https://user-images.githubusercontent.com/47939832/111865936-58608480-89ad-11eb-897a-d8cc03937987.png)
        
        위 디렉토리에서 helm install --name prometheus --namespace monitoring . 명령어를 입력합니다.
        
        시간이 꽤 걸리니 몇분 기다리셔야 합니다.
        
    - 정상적으로 빼포되었는지 확인
        
        ~~~
        kubectl get all -n monitoring
        ubuntu@kfc-dj:~/kfcHelm/charts/myhelm/prometheus-operator$ kubectl get all -n monitoring
        NAME                                                         READY   STATUS    RESTARTS   AGE
        pod/alertmanager-prometheus-prometheus-oper-alertmanager-0   2/2     Running   0          32d
        pod/prometheus-grafana-69fdcbf946-w7sm8                      2/2     Running   0          33d
        pod/prometheus-kube-state-metrics-6bc5d5dc67-p9ntg           1/1     Running   2          33d
        pod/prometheus-prometheus-node-exporter-6bddh                1/1     Running   2          56d
        pod/prometheus-prometheus-node-exporter-8wrss                1/1     Running   2          56d
        pod/prometheus-prometheus-node-exporter-b7hkb                1/1     Running   2          56d
        pod/prometheus-prometheus-node-exporter-hqcsx                1/1     Running   2          48d
        pod/prometheus-prometheus-node-exporter-m5ngz                1/1     Running   1          56d
        pod/prometheus-prometheus-node-exporter-rtz77                1/1     Running   2          56d
        pod/prometheus-prometheus-oper-operator-79cd655689-f6n5k     2/2     Running   1          33d
        pod/prometheus-prometheus-prometheus-oper-prometheus-0       3/3     Running   0          32d

        NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
        service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   56d
        service/prometheus-grafana                        NodePort    10.102.125.63    <none>        80:30010/TCP                 56d
        service/prometheus-kube-state-metrics             ClusterIP   10.109.54.193    <none>        8080/TCP                     56d
        service/prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     56d
        service/prometheus-prometheus-node-exporter       ClusterIP   10.103.214.146   <none>        9100/TCP                     56d
        service/prometheus-prometheus-oper-alertmanager   ClusterIP   10.103.86.243    <none>        9093/TCP                     56d
        service/prometheus-prometheus-oper-operator       ClusterIP   10.98.119.171    <none>        8080/TCP,443/TCP             56d
        service/prometheus-prometheus-oper-prometheus     NodePort    10.108.25.24     <none>        9090:30011/TCP               56d

        NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
        daemonset.apps/prometheus-prometheus-node-exporter   6         6         6       6            6           <none>          56d

        NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/prometheus-grafana                    1/1     1            1           56d
        deployment.apps/prometheus-kube-state-metrics         1/1     1            1           56d
        deployment.apps/prometheus-prometheus-oper-operator   1/1     1            1           56d

        NAME                                                             DESIRED   CURRENT   READY   AGE
        replicaset.apps/prometheus-grafana-69fdcbf946                    1         1         1       56d
        replicaset.apps/prometheus-kube-state-metrics-6bc5d5dc67         1         1         1       56d
        replicaset.apps/prometheus-prometheus-oper-operator-79cd655689   1         1         1       56d

        NAME                                                                    READY   AGE
        statefulset.apps/alertmanager-prometheus-prometheus-oper-alertmanager   1/1     56d
        statefulset.apps/prometheus-prometheus-prometheus-oper-prometheus       1/1     56d
        ubuntu@kfc-dj:~/kfcHelm/charts/myhelm/prometheus-operator$
        ~~~
        
        위와같이 정상 배포되었는지 확인할 수 있습니다.
        
    - prometheus-node-exporter는 모든 노드에 배포되어 있으며 나머지 패키지들은 kfc-dj2 즉, key=monitoring에 배포 되어 있어야함.
        
        ~~~
        ubuntu@kfc-dj:~/kfcHelm/charts/myhelm/prometheus-operator$ kubectl get pod -n monitoring -o wide
        NAME                                                     READY   STATUS    RESTARTS   AGE   IP               NODE               NOMINATED NODE   READINESS GATES
        alertmanager-prometheus-prometheus-oper-alertmanager-0   2/2     Running   0          32d   10.234.211.87    kfc-dj2            <none>           <none>
        prometheus-grafana-69fdcbf946-w7sm8                      2/2     Running   0          33d   10.234.211.84    kfc-dj2            <none>           <none>
        prometheus-kube-state-metrics-6bc5d5dc67-p9ntg           1/1     Running   2          33d   10.234.211.85    kfc-dj2            <none>           <none>
        prometheus-prometheus-node-exporter-6bddh                1/1     Running   2          56d   103.22.222.245   kfc-dj             <none>           <none>
        prometheus-prometheus-node-exporter-8wrss                1/1     Running   2          56d   103.22.222.247   kfc-dj3            <none>           <none>
        prometheus-prometheus-node-exporter-b7hkb                1/1     Running   2          56d   103.22.222.246   kfc-dj2            <none>           <none>
        prometheus-prometheus-node-exporter-hqcsx                1/1     Running   2          48d   116.89.190.210   ssu-nc-in-gist-1   <none>           <none>
        prometheus-prometheus-node-exporter-m5ngz                1/1     Running   1          56d   116.89.189.54    master             <none>           <none>
        prometheus-prometheus-node-exporter-rtz77                1/1     Running   2          56d   116.89.189.21    worker-1           <none>           <none>
        prometheus-prometheus-oper-operator-79cd655689-f6n5k     2/2     Running   1          33d   10.234.211.86    kfc-dj2            <none>           <none>
        prometheus-prometheus-prometheus-oper-prometheus-0       3/3     Running   0          32d   10.234.211.88    kfc-dj2            <none>           <none>
        ubuntu@kfc-dj:~/kfcHelm/charts/myhelm/prometheus-operator$
        ~~~
        
        ![8](https://user-images.githubusercontent.com/47939832/111865939-58f91b00-89ad-11eb-91fb-23138d552d3e.png)
        
    - grafana와 prometheus의 NodePort를 구성합니다. 이때, nodeport 번호는 중복되지 않도록 합니다.
        
        ![9](https://user-images.githubusercontent.com/47939832/111865940-5991b180-89ad-11eb-90ed-c229691c4be3.png)
        
        아래 명령어를 입력 후, 수정합니다.
        
        > kubectl edit svc prometheus-grafana -n monitoring
        > 
        
        ![91](https://user-images.githubusercontent.com/47939832/111865941-5991b180-89ad-11eb-951e-0c9095bbd955.png)
        
        > kubectl edit svc prometheu-prometheus-oper-prometheus -n monitoring
        > 
        
        ![92](https://user-images.githubusercontent.com/47939832/111865942-5a2a4800-89ad-11eb-9423-bb632ce3a907.png)
        
        위와 같이 svc를 수정 후, 편집한 서비스를 재확인해 보면
        
        ![93](https://user-images.githubusercontent.com/47939832/111865943-5a2a4800-89ad-11eb-8802-47b06e72f62a.png)
        
        다음과 같음을 알 수 있습니다.
        
    - grafana 확인
        
        기본 설정 id와 password는
        
        'admin', 'prom-operator' 입니다.
        
        ![94](https://user-images.githubusercontent.com/47939832/111865944-5ac2de80-89ad-11eb-9d90-c91930b7dcbe.png)
        
        로그인 후 화면은 아래와 같습니다.
        
        ![95](https://user-images.githubusercontent.com/47939832/111865945-5ac2de80-89ad-11eb-96c3-0b1ebbba7e99.png)
        
        manage 메뉴로 들어갑니다.
        
        ![96](https://user-images.githubusercontent.com/47939832/111865946-5b5b7500-89ad-11eb-80d6-9faec4a8c182.png)
        
        다음과 같이 그라파나가 관리하고 있는 클러스터의 상태를 확인해 볼 수 있습니다.
        
        ![97](https://user-images.githubusercontent.com/47939832/111865947-5b5b7500-89ad-11eb-9396-b0e64ebf2471.png)
        
        ![98](https://user-images.githubusercontent.com/47939832/111865948-5bf40b80-89ad-11eb-8b14-227b407e715a.png)
        
        마찬가지로 prometheus와 prometheus alertmanager에도 접근할 수 있습니다.
        
        만약 helm values.yaml을 수정하려면 helm upgrade 명령어를 사용하시면 됩니다. 또한, 만약 release가 잘못되었다면 helm rollback 명령어를 통해 롤백이 가능합니다.
