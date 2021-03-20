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
