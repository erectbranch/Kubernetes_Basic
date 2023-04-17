# 1 slurm 정리

> [SLURM 가이드](https://thisisntnathan.github.io/dftCourse/ShortCourse/slurm.html)

> [SLURM 스케줄러를 이용한 작업 제출 및 관리](https://dandyrilla.github.io/2017-04-11/jobsched-slurm/)

**SLURM**(Simple Linux Utility for Resource Management)은 workload manager의 일종으로, 대규모 multi-user systems에서 computation resources 할당을 control하는 데 사용된다.

slurm의 필수 구성 요소로는 control daemon이자 job scheduling을 맡는 `slurmctld`와 compute job의 launch를 담당하는 `slurmd`가 있다.

- `slurmctld`를 management servers라고 부른다.

- `slurmd`에 의해 running되는 nodes를 compute nodes라고 한다.

---

## 1.1 daemon

**daemon**은 system 기능을 제공하거나 백그라운드에서 항시 실행되는 program을 의미한다. 네트워크 요청이나 하드웨어 동작 등 여러 기능을 담당하며 다양하게 사용되어 **system process**라고도 불린다. 예를 들면 telnet, httpd, mysql, sshd처럼 백그라운드에서 장시간 돌아가는 process가 있다.

> Window의 service와 같은 개념으로 process 형태로 실행되며, deamon임을 나타내기 위해 process 이름에 'd'가 뒤에 붙는다.(syslogd 등)

- **standalone**: httpd 등 메일 서버, 웹 서버처럼 항상 service가 가능하게 수행되며, 혼자서 request를 받아서 처리하는 daemon을 의미한다. 

  > 특성상 memory 사용량이 많을 수밖에 없어서, client request가 많지 않은 network service에서는 비효율적이다.

- **super daemon**(xinetd): client 등에서 들어온 request에 따라 daemon을 실행시키는 방식이다.

  > xinetd도 daemon이므로 system이 켜질 때 standalone으로 실행된다. 이후 request에 따라 다른 daemon을 불러들여 실행시키게 된다.

background program은 일반적으로 user와 상호작용하기 위해 terminal을 갖지만, daemon은 특별한 경우가 아니면 user와 상호작용할 필요 없이 아무도 모르게 실행되어야 하므로 terminal을 갖지 않는다.

> 'daemon'이라는 단어가 갖는 의미(귀신)을 생각하자. 참고로 원래 두문자어는 아니지만, disk and execution monitor로 뜻을 맞춰서 부르기도 한다.

> [프로세스 요약](https://4legs-study.tistory.com/37)

또한 대부분의 OS는 process를 구분하기 위해 각 process에 PID(Process Identifier)를 부여하였다. process는 parent process와, 이 parent process가 새롭게 생성(**fork()**)한 child process를 가지는 구조로 이루어졌는데, daemon은 PPID(Process Parent Identifier)를 갖지 않는다.(정확히는 PPID가 1로 세팅된다.) 즉, daemon은 다른 process의 child process가 아니며 어떤 process의 영향도 받지 않음을 의미한다. 

이런 특징 때문에 OS가 종료되기 전까지 계속 살아있게 된다. 위 내용을 바탕으로 daemon process의 특징을 요약하면 다음과 같다.

- terminal을 갖지 않는다.

- PPID가 1인 process다.

process가 daemon의 특성을 갖도록 만드는 방법이 있다.

1. fork()로 child process를 만든다.(PPID는 parent process의 PID가 된다.)

2. parent process를 종료(**exit()**)한다. 

  - 기존의 child process는 고아 프로세스가 된다. 고아 프로세스는 PID가 1인 init process를 PPID로 갖게 된다.

3. **setsid**를 이용해서 새 세션을 만들고 child process를 그 세션의 leader로 만든다.

  - 이렇게 하면 child process는 새로운 세션의 leader가 되고, PPID는 1이 된다.(terminal(tty)를 갖지 않는 process가 된다.)

4. **chdir**을 이용해서 현재 작업 디렉토리를 root directory로 변경한다.

  - 이렇게 하면 daemon이 사용하는 파일 시스템을 unmount할 수 있다.

---

## 1.2 allocate resources

내 job을 실행하기 위해서는 다음과 같은 과정을 거쳐야 한다.

1. slurm daemon에게 job에 allocate할 resource를 요청한다.

2. script를 가져와서 compute power가 얼마나 필요한지 계산한다.

3. nodes/memory가 available하면 해당 nodes/memory에서 script를 실행한다.

    - 이때 available한 resource가 없다면, 다시 available하게 될 때까지 queue에서 대기한다.

> [SLURM Documentation](https://slurm.schedmd.com/)

> [SLURM command cheat sheet](https://slurm.schedmd.com/pdfs/summary.pdf)

이를 위한 여러 command은 공식 Documentation을 확인하면 알 수 있다. 지금은 이 중에서도 가장 많이 사용하는 command를 살펴보자.

---

### 1.2.1 Gathering information

`sinfo` command를 사용하면 computing nodes의 상태를 확인할 수 있다. 다음은 여러 cpu, gpu partition으로 나뉜 서버에서 sinfo command를 입력한 결과이다.

- **partition**: compute nodes를 그룹화한 것이다.

- `--user`, `--partition`, `--state` 등 다양한 option을 사용해서 filter를 적용할 수 있다.

```bash
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
gpu1         up   infinite      7    mix n[003-006,009-011]
gpu1         up   infinite      4  alloc n[001-002,007-008]
gpu1         up   infinite      2   idle n[012-013]
cpu1         up   infinite     10    mix n[014,016-024]
cpu1         up   infinite     25   idle n[015,025-048]
hgx          up   infinite      1   idle n050
gpu2         up   infinite      3    mix n[051-052,056]
gpu2         up   infinite     33   idle n[053-055,057-086]
cpu2*        up   infinite      2  alloc n[087,091]
cpu2*        up   infinite     12   idle n[088-090,092-100]
```

- mix: job이 실행 중이지만, available한 resource가 있다.

- idle

- alloc: job을 위해 fully allocated된 상태다.

- down

위 예제에서는 사용하는 gpu, cpu에 따라 partition을 만들었지만, ownership이나 usage(large/small jobs, post-processing, data visualization 등)로 구분해서 만들 수도 있다.

좀 더 세부적으로 살펴보기 위해 `pestat` command를 사용할 수 있다.

- 'F' 옵션을 주면 flagged된 node만 print할 수 있다.

```bash
$ pestat
Hostname       Partition     Node Num_CPU  CPUload  Memsize  Freemem  Joblist
                            State Use/Tot              (MB)     (MB)  JobId User ...
    n001            gpu1    alloc  48  48    0.86*   768000   377406  10916860 user1 10917036 user2  
    n002            gpu1    alloc  48  48    1.60*   768000    51286* 10917038 user2 10917037 user2  
    ...
```

또한 `squeue` command를 이용하면 job queue도 확인할 수 있다.
```bash
$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
          10917037      gpu1 chem_test   user2  R    4:33:53      1 n002
          10917036      gpu1 chem_test   user2  R    4:33:59      1 n001
          ...
```

---

### 1.2.2 Submitting jobs

- `sbatch {script}` command를 이용하면 job을 submit할 수 있다.

- `scancel {job ID}` command를 이용해서 job을 취소할 수 있다.

---