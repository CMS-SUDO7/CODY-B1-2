# **🚀 Agent App Troubleshooting Guide**

백엔드 시스템 및 에이전트 애플리케이션의 \*\*3대 치명적 장애(OOM, CPU Spike, Deadlock)\*\*를 의도적으로 재현하고, 리눅스 OS 레벨에서 이를 진단 및 해결한 트러블슈팅 사후 분석(Post-Mortem) 리포지토리입니다.

💡 상세한 장애 분석 보고서와 관제 데이터는 본 리포지토리 issues 탭에서 확인 가능합니다.

## **🛠 1\. 실행 환경 및 컨트롤 패널**

애플리케이션(agent-leak-app-x86)의 동작 방식은 부팅 스크립트인 run.sh의 환경변수 조작을 통해 완벽하게 제어됩니다. 아래는 애플리케이션을 안정적으로 구동시키는 **Safe Mode** 기준의 스크립트입니다.

```
#!/bin/bash

# [디렉토리 셋업]
export AGENT_HOME="/home/agent-admin/agent-app"
export AGENT_LOG_DIR="$AGENT_HOME/logs"
mkdir -p "$AGENT_LOG_DIR"

# [환경변수 셋업 - 장애 제어 트리거] 
export AGENT_PORT=15034
export MEMORY_LIMIT=512             # 메모리 할당량 (누수 방어)
export CPU_MAX_OCCUPY=10            # CPU 제한 (Spike 방어)
export MULTI_THREAD_ENABLE="false"  # 멀티 스레드 (Deadlock 방어)

echo "✅ [run.sh] 환경변수 세팅 완료"
chmod +x ./agent-leak-app-x86
./agent-leak-app-x86
```

## **💥 2\. 장애 재현 (Reproduction) 시나리오**

run.sh의 환경변수를 수정하여 아래 3가지 장애를 의도적으로 유발하고 시스템의 반응을 관측할 수 있습니다.

### **🚰 Case 1: Out Of Memory (OOM) / Memory Leak**

객체가 메모리에서 해제되지 않고 지속적으로 누수되어, OS 커널(OOM Killer) 또는 내부 보호 로직에 의해 프로세스가 강제 종료되는 현상입니다.

* **트리거:** export MEMORY\_LIMIT=50 (메모리 제한을 극단적으로 낮춤)  
* **예상 결과:** 프로세스 RSS 메모리가 지속 우상향하다가 임계치 초과로 강제 종료됨.  
* **관측 명령:** watch \-n 1 \-d ./monitor.sh (메모리 팽창 실시간 하이라이트 관측)

### **🔥 Case 2: CPU Spike (자원 과점유)**

특정 프로세스가 무한 루프나 비효율적인 연산으로 CPU를 독식하여, 시스템 Run Queue에 병목을 유발하는 현상입니다.

* **트리거:** export CPU\_MAX\_OCCUPY=100 (CPU 점유율 제한 해제)  
* **예상 결과:** CPU 사용률이 급증하며, 내부 Watchdog 로직에 의해 SIGTERM 강제 종료됨.  
* **관측 명령:** top \-b \-d 0.1 \-n 100 \-p \[PID\] \> trace.log (0.1초 단위 Micro-burst 배치 캡처)

### **🔒 Case 3: Deadlock (교착 상태)**

다중 스레드 환경에서 자원(Lock) 획득 순서가 꼬여, 스레드들이 서로를 무한정 기다리는 순환 대기(Circular Wait) 현상입니다.

* **트리거:** export MULTI\_THREAD\_ENABLE="True" (다중 스레드 활성화)  
* **예상 결과:** 앱이 종료되지는 않으나 로그 출력이 멈추고 연산이 완전히 정지됨(Hang).  
* **관측 명령:** top \-H \-p \[PID\] (내부 스레드들이 0.0% CPU로 S (Sleep) 상태에 빠진 것 확인)

## **🔎 3\. 핵심 리눅스 트러블슈팅 명령어 (Cheat Sheet)**

장애 진단 및 원인 규명을 위해 사용된 필수 OS 레벨 관측 도구 모음입니다.

### **📸 프로세스 상태 진단 (Snapshot)**

|

| **명령어** | **용도** |

| pgrep \-f agent-leak | 애플리케이션의 PID만 빠르게 추출 |

| ps \-ef | grep agent-leak | 프로세스의 생존 여부 및 실행 인자(부모/자식 구조) 확인 |

| pstree \-p \[PID\] | 쉘 스크립트와 실행 파일 간의 프로세스 트리(계층 구조) 시각화 |

### **📹 실시간 자원 모니터링 (Real-time)**

| **명령어** | **용도** |

| top \-p \[PID\] | 특정 프로세스의 실시간 점유율 감시 (키보드 P CPU 정렬, M 메모리 정렬) |

| watch \-n 1 \-d ./monitor.sh | 커스텀 관제 스크립트를 1초마다 갱신하며 수치 변화 하이라이트(-d) |

### **🔬 스레드 및 미세 병목 추적 (Deep Dive)**

| **명령어** | **용도** |

| top \-H \-p \[PID\] | 프로세스 내부의 스레드들을 분리하여 교착상태(S 상태) 스레드 식별 |

| top \-b \-d 0.1 \-n 100 \-p \[PID\] | 화면 새로고침 없이 0.1초 단위 100번 스냅샷을 텍스트로 기록(Dump) |

| tail \-f logs/agent\_app.log | 애플리케이션 내부 다잉 메시지(BLOCKED 등) 실시간 스트리밍 |

sudo cp /Users/herebattle6145/Downloads/agent-app-leak/agent-leak-app-x86 /home/agent-admin/agent-app/

sudo chmod +x /home/agent-admin/agent-app/agent-leak-app-x86

chmod +x run.sh
