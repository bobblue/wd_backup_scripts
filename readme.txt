# 백업
./all-backup-restore.sh backup -n zen --skip-components elastic

# 복원 (같은 옵션을 똑같이 줘야 elastic.backup을 찾지 않음)
./all-backup-restore.sh restore -f watson-discovery_20260419.backup --skip-components elastic
