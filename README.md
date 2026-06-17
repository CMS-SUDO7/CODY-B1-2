# CODY-B1-2

1. 메모리50 cpu10 멀티스레드 false (OOM) vs 메모리512 cpu10 멀티스레드 false (정상)
2. 메모리512 cpu100 멀티스레드 false (CPU-Latency->watchdog) vs 메모리512 cpu10 멀티스레드 false (정상)



sudo cp /Users/herebattle6145/Downloads/agent-app-leak/agent-leak-app-x86 /home/agent-admin/agent-app/

sudo chmod +x /home/agent-admin/agent-app/agent-leak-app-x86

chmod +x run.sh

```
cat << 'EOF' > /home/agent-admin/agent-app/run.sh
#!/bin/bash

export AGENT_HOME="/home/agent-admin/agent-app"
export AGENT_UPLOAD_DIR="$AGENT_HOME/upload_files"
export AGENT_KEY_PATH="$AGENT_HOME/api_keys"
export AGENT_LOG_DIR="$AGENT_HOME/logs"

mkdir -p "$AGENT_UPLOAD_DIR"
mkdir -p "$AGENT_KEY_PATH"
mkdir -p "$AGENT_LOG_DIR"

echo "agent_api_key_test" > "$AGENT_KEY_PATH/secret.key"
chmod 600 "$AGENT_KEY_PATH/secret.key"

# [환경변수 셋업] 
export AGENT_PORT=15034
export MEMORY_LIMIT=512             # 메모리 넉넉하게
export CPU_MAX_OCCUPY=10            # ★ CPU 제한을 10%로 빡빡하게
export MULTI_THREAD_ENABLE="false"  # ★ 싱글 스레드 모드 (데드락 회피)

echo "✅ [run.sh] 환경변수 세팅 완료 (MEMORY: 512MB / CPU: 10% / THREAD: false)"
echo "🚀 애플리케이션을 실행합니다..."

chmod +x ./agent-leak-app-x86
./agent-leak-app-x86
EOF
```
