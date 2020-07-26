# CPU with an integrated FPGA
Intel의, [CPU와 FPGA가 결합된 칩](https://www.nextplatform.com/2018/05/24/a-peek-inside-that-intel-xeon-fpga-hybrid-chip/)으로 하드웨어 가속을 하는 법을 알아보자.



> 이 튜토리얼은 intel의 [튜토리얼](https://wiki.intel-research.net/FPGA.html)에 기반해 작성됐다.

시작하기 앞서, 몇몇 용어들을 확인하고 넘어가자.

- FPGA(Field-Programmable Gate Array)
- AFU(Accelerator Functional Unit)
- ASE(AFU Simulation Environment)
- OPAE(Open Programmable Acceleration Engine)
- CCI-P(Core Cache Interface)
- HARP(Hardware Accelerator Research Program)




-
-
-
-
-
-

## Configuring a build environment

먼저, 배정받은 서버(호스트 : ssh-iam.intel-research.net)에 ssh로 연결한다. FPGA를 사용하는 데 필요한 파일들은 /export/fpga 위치에 저장돼 있다.(OPAE 등)

다음으로, 해당 FPGA class를 [이 표](https://wiki.intel-research.net/FPGA.html#id2)에서 찾아서, 다음 command의 \<fpga-class> 자리에 넣고 실행한다.

```shell
source /export/fpga/bin/setup-fpga-env <fpga-class>
```

단, 이 경우 OPAE SDK 와 BBB는 공용 버전을 따라가는데, OPAE SDK의 개인 복사본을 써야 하는 경우는 위 command에 앞서 OPET_INSTALL_PATH를 지정해야 한다. OPAE SDK 설치 경로를 넣어서, 다음을 실행하면 된다. (혹시 OPAE SDK의 source code가 필요하면, [여기](https://github.com/OPAE/opae-sdk)에 있다.)

```shell
# 기본 버전이 아닌 OPAE SDK를 고른다.
export OPAE_INSTALL_PATH=<path to private OPAE SDK installation>
source /export/fpga/bin/setup-fpga-env <fpga-class>
```


## Working with RTL

#### Synthesizing RTL designs

Quartus와 OPAE는 /export/fpga 에 이미 설치되어 있고, 위에서 setup-fpga-env 가 source 되었을 때 환경변수 PATH에 들어갔다.

이제 RTL 을 synthesize할 단계이다. 공용 머신인 만큼, queue에 필요한 작업을 넣음으로써 모든 유저가 공평하게 컴퓨팅 자원을 얻게 되는데, 그 과정에 쓰이는 명령어가 qsub이다. qsub -q \<queue-name> \<job-script> 와 같이 쓰며, 자세한 내용은 [여기](https://wiki.intel-research.net/Introduction.html#how-to-submit-a-task)에서 볼 수 있다.

지금은 qsub-synth 명령어를 이용해 synthesize 태스크를 queue에 넣어 RTL을 synthesize 해볼 것이다.

다음은 BBB git repository를 clone해 tutorial중 하나인 01_hello_world 를 현재 설정된 FPGA상에서 synthesize하는 과정이다.


```shell
git clone https://github.com/OPAE/intel-fpga-bbb
cd intel-fpga-bbb/samples/tutorial/01_hello_world/hw
# Quartus 빌드 영역을 설정한다.
afu_synth_setup -s rtl/sources.txt build_fpga
cd build_fpga
# 앞서 설명한 vlab batch queue에서 Quartus를 실행한다.
qsub-synth
# 빌드 과정을 실시간으로 볼 수 있는 로그 파일을 만든다.(optional)
tail -f build.log
```

이 과정은 시간이 조금 걸리는데, shell에 ```qstat``` 명령어를 쳐서 잘 실행되고 있는지 확인할 수 있다. (자신의 username을 기반으로 찾으면 된다.)

환경 설정과 synthesis 과정에 대해 더 자세히 알고 싶으면 위 git을 [참고](https://github.com/OPAE/intel-fpga-bbb/tree/master/samples/tutorial)하면 좋다.



#### Executing on FPGAs


```shell
# 설정된 class의 FPAG 시스템에 해당하는 shell을 연다.
qsub-fpga
# 새로운 shell에서 위의 qsub-synth가 실행되던 위치로 돌아간다.
cd $PBS_O_WORKDIR
# 해당 image를 FPAG에 load한다.
fpgaconf cci_hello.gbs
# 해당 프로그램을 컴파일한다.
cd ../../sw
make
# 컴파일된 프로그램을 실행한다.
./cci_hello
```

위와 같이, ``` qsub-fpga ```를 실행하면, FPGA를 사용할 수 있는 새로운 shell이 열리고, 이때 이 스크립트는 위에서 설정된 FPGA class를 고른다. 또, -V 옵션이 자동으로 qsub에 적용돼서, 기존의 환경 설정들도 모두 새로운 shell로 넘어간다.

그리고 적절히 컴파일하고 실행하면, "Hello World!"라는 문구를 볼 수 있을 것이다.

``` qsub-fpga ```
를 실행할 때마다 작업 directory가 바뀌는 것이 번거로우면, 다음 코드를 ~/.bashrc에 추가하고 ``` source ~/.bashrc ```를 shell에 실행시키면 새로운 shell도 같은 작업 directory에서 실행될 것이다.

```shell
if [ "$PBS_ENVIRONMENT" == "PBS_INTERACTIVE" ] && [ "$PBS_O_WORKDIR" != "" ] &&
   [ "$PBS_O_HOME" == "$PWD" ] && [ "$PBS_O_WORKDIR" != "$PWD" ]; then
  echo "Moving to PBS workdir: $PBS_O_WORKDIR"
  cd "$PBS_O_WORKDIR"
fi
```


이로써, 위에서 받은 첫번째 tutorial이 정상적으로 실행됨을 확인했다.






#### Simulating designs with ASE

다음은, 같은 tutorial을 실행하되, ASE(AFU Simulation Environment)를 이용하여 RTL 시뮬레이션을 해보자.

먼저, 다음 명령어를 shell에서 치자.

```shell
qsub-sim
```

```shell
qsub-sim
# 위에서 작업하던 directory로 이동한다.
cd <path to intel-fpga-bbb/samples/tutorial/01_hello_world/hw>
# simulation build directory를 만든다.
afu_sim_setup -s rtl/sources.txt build_sim
# tmux로 terminal을 2개로 나눈다.
tmux
^b"
# 컴파일한 뒤 RTL simulator를 실행한다.
cd build_sim
make
make sim
# 다른 terminal로 이동한다.
^bo
cd ../sw
make
# 위에서 make sim의 실행 결과 중 export ASE_WORKDIR=<path> 부분을 복사해 다음과 같이 실행한다.
export ASE_WORKDIR=<path to intel-fpga-bbb/samples/tutorial/01_hello_world/hw/build_sim/work>
with_ase ./cci_hello
```


#### AAL legacy Software
