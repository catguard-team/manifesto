# 누출 정리 플레이북 (Leak Cleanup)

> [`SAFETY.md` §1](./SAFETY.md#1-절대-금지--개인정보)이 "이미 들어가 있는 게 발견되면 즉시 제거 + git history 정리"라고 명령하면, 이 문서대로 한다.

---

## 0. 어떤 상황에 이 문서를 펴는가

다음 중 하나라도 git에 들어갔다면:

- 새끼고양이 실명·학교·주소·연락처·생년월일·기기 시리얼
- 보호자 동의 없는 식별 가능 사진/음성
- API 키, 토큰, 비밀번호, 인증서 (GitHub PAT, AWS 키, OAuth secret 등)
- 자경단원 본인의 비공개 정보를 본인이 동의 없이 올린 경우

→ **즉시 이 플레이북 시작.** 단순 `git rm` + commit은 부족하다 (히스토리에 남는다).

---

## 1. 0단계 — 일단 멈춰라 (5분 안에)

```bash
# 1. 추가 push 금지 — 다른 자경단원에게 디스코드 #그루밍 알림
echo "🚨 leak-cleanup in progress on <repo>" | (디스코드에 붙여넣기)

# 2. CI/배포 일시 중단 (해당되면)
# 3. 누출된 정보가 토큰류면 즉시 폐기 (revoke). 정리는 그 다음.
```

**왜 폐기 먼저인가**: git history를 청소해도 GitHub의 archive·다른 사람의 fork·웹 캐시·CDN에 이미 퍼졌을 수 있다. **노출된 토큰은 이미 노출됐다고 간주**하고 즉시 무효화.

### 토큰 즉시 폐기 체크리스트

| 토큰 | 폐기 방법 |
|------|-----------|
| GitHub PAT | github.com/settings/tokens → 해당 토큰 **Delete** |
| GitHub fine-grained PAT | github.com/settings/personal-access-tokens → **Revoke** |
| 레포 secret (Actions) | 본인 레포 Settings → Secrets → 삭제 후 재생성 |
| AWS Access Key | IAM → 키 비활성화 → 삭제 → 새 키 발급 |
| OAuth Client Secret | 해당 provider 콘솔에서 rotate |
| DB 비밀번호 | DB에서 즉시 변경 + 의존 서비스 재배포 |

---

## 2. 1단계 — 영향 범위 파악

```bash
# 어떤 커밋·파일·브랜치·태그에 들어갔는지
git log --all --full-history -p -S "노출된문자열" | head -100

# 모든 ref에서 검색
git grep "노출된문자열" $(git rev-list --all) | head -20
```

기록할 것:
- 첫 등장 커밋 SHA
- 영향받은 파일 목록
- public 노출 시간 (push 시점 ~ 지금)
- fork·미러 존재 여부 (`gh api repos/<owner>/<repo>/forks`)

---

## 3. 2단계 — 백업 (필수, 생략 금지)

BFG는 히스토리를 **돌이킬 수 없게** 다시 씁니다. 사고 시 복구 불가능. 반드시 백업.

```bash
# 미러 클론으로 모든 ref 백업
cd /tmp
git clone --mirror https://github.com/<owner>/<repo>.git <repo>-backup-$(date +%Y%m%d-%H%M)
ls <repo>-backup-*  # 디렉토리 확인
```

이 백업은 **로컬에서만 보관**, 절대 push 하지 말 것 (노출 정보가 아직 그대로).

---

## 4. 3단계 — BFG Repo-Cleaner 설치·실행

### 설치 (macOS)

```bash
brew install bfg
bfg --version  # v1.14.0+
```

또는 jar 직접:

```bash
curl -L -o /tmp/bfg.jar \
  https://repo1.maven.org/maven2/com/madgag/bfg/1.14.0/bfg-1.14.0.jar
alias bfg='java -jar /tmp/bfg.jar'
```

### 실행 — 케이스별

작업 디렉토리는 **새로 받은 mirror clone**에서.

```bash
cd /tmp
git clone --mirror https://github.com/<owner>/<repo>.git
cd <repo>.git
```

#### A) 특정 문자열 (토큰·실명·전화번호) 제거

```bash
# 제거할 문자열을 파일에 한 줄씩
cat > /tmp/secrets.txt <<'EOF'
ghp_AbCd1234ExampleTokenString
홍길동
010-1234-5678
EOF

bfg --replace-text /tmp/secrets.txt
# 결과: 모든 히스토리에서 해당 문자열이 ***REMOVED***로 치환됨
```

#### B) 특정 파일 통째 삭제 (히스토리 전체에서)

```bash
# 파일명 (모든 경로의 동일 이름)
bfg --delete-files "id_rsa"
bfg --delete-files "credentials.json"

# 폴더 통째
bfg --delete-folders "private-photos" --no-blob-protection
```

#### C) 큰 파일 제거 (실수로 올린 사진/영상)

```bash
bfg --strip-blobs-bigger-than 10M
```

> ⚠️ BFG는 기본적으로 **현재 HEAD에 있는 파일은 보호**합니다. 현재 HEAD에 노출 정보가 남아있으면 먼저 일반 commit으로 제거 후 push, 그다음 BFG.

### 정리 + reflog 만료

```bash
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

---

## 5. 4단계 — Force-push (이번엔 mirror push)

```bash
# 모든 ref 한 번에 강제 갱신
git push --force --mirror
```

이게 일반 force-push와 다른 점: `--mirror`는 **모든 브랜치·태그·refs**를 한꺼번에 source 상태로 만든다. 누락 ref가 없도록.

---

## 6. 5단계 — GitHub 측 캐시 삭제 (중요)

force-push만으로는 GitHub에 잔존하는 캐시가 안 지워집니다.

### 6-1. dangling commit 즉시 GC 요청

```bash
gh api -X POST repos/<owner>/<repo>/dispatches -f event_type=cleanup 2>/dev/null || true
# 위는 이벤트만; 실제 GitHub GC는 사람이 요청해야 함
```

GitHub 지원에 메일:
- 주소: `support@github.com`
- 제목: `Sensitive data leak — request for cached view removal`
- 본문: 레포 URL, 노출된 커밋 SHA, 즉시 정리한 내역, 캐시 페이지 GC 요청
- 참고: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository

### 6-2. fork 처리

```bash
gh api repos/<owner>/<repo>/forks --jq '.[] | .full_name'
```

→ 각 fork 소유자에게 디스코드/이슈로 정리 요청. 응답 없으면 GitHub 지원에 함께 신고.

### 6-3. 자경단 측 정리

자경단은 자체 레포 외에는 사본을 두지 않으므로 (다른 사람 레포는 [링크 등재](./OPERATIONS.md#11)만), 누출이 외부 자경단원 레포에서 발생한 경우 자경단이 BFG 작업할 사본이 없다 — 원작자가 본인 레포에서 정리하면 끝. 자경단은 인덱스 링크가 살아있는지만 점검.

자체 레포(manifesto, .github 등)에서 누출이 발생한 경우는 위 5단계가 그대로 적용된다 (`catguard-team/<repo>`에 직접 force-push --mirror).

---

## 7. 6단계 — 알림 + 회고

[`SAFETY.md` §7 사고 발생 시](./SAFETY.md#7-사고-발생-시) 절차 동시 진행:

1. ✅ 단장에게 알림 (디스코드 DM)
2. ✅ 새끼고양이/보호자에게 사실대로 알림 (해당되면)
3. ✅ 사용자/자경단원에게 토큰 폐기·재로그인 안내
4. ✅ [`CHANGELOG.md`](./CHANGELOG.md)에 회고 1줄 기록 (식별 정보 제거 형태)

회고 예시:
```
## YYYY-MM-DD — leak cleanup
- repo: <repo>
- type: GitHub PAT (revoked at HH:MM)
- 노출 시간: 약 N분 (push HH:MM ~ revoke HH:MM)
- 영향: 해당 PAT는 catguard-team/<repo> 1개 레포 push 권한
- 조치: 토큰 폐기 → BFG 정리 → force-push → 새 PAT 발급
- 재발 방지: pre-commit hook 도입 검토 (예방 §아래)
```

---

## 8. 예방 — 다음에 또 이 플레이북을 펴지 않기 위해

### 8-1. pre-commit secret scan

```bash
brew install gitleaks
cd <repo>
cat > .pre-commit-config.yaml <<'YAML'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
YAML
pre-commit install
```

→ commit 시 토큰 패턴이 자동 차단된다.

### 8-2. GitHub Secret Scanning

- Settings → Code security → Secret scanning **enable**
- Push protection **enable** (push 자체를 차단)

### 8-3. 개인정보 점검 체크리스트 (PR 자체 검수)

PR 올리기 전 5초:

- [ ] 실명 / 전화번호 / 이메일 / 주소가 코드·테스트·픽스처에 박혀있지 않은가
- [ ] `.env`, `secrets.json`, `id_rsa`, `*.pem`이 `.gitignore`에 있는가
- [ ] 스크린샷에 식별 가능한 화면이 있지 않은가
- [ ] 커밋 메시지에 토큰·이름이 들어가지 않았는가

---

## 9. 자주 하는 실수

| 실수 | 결과 | 올바른 처리 |
|------|------|-------------|
| `git rm secret.txt && git commit` 후 끝 | 히스토리에 남음. **누출 그대로** | BFG로 히스토리에서 제거 |
| 일반 working clone에서 BFG 실행 | 일부 ref 누락 가능 | `--mirror` clone에서 실행 |
| force-push만 하고 토큰은 그대로 둠 | 이미 새어나간 토큰 유효 | **토큰 폐기 먼저** |
| BFG 후 `git gc` 안 함 | dangling 객체 남음 | `git reflog expire --expire=now --all && git gc --prune=now --aggressive` |
| fork·미러 무시 | 다른 사본에 누출 잔존 | fork 목록 확인 + 미러 동시 정리 |

---

## 10. 참고 자료

- BFG 공식: https://rtyley.github.io/bfg-repo-cleaner/
- GitHub 공식 가이드: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository
- gitleaks: https://github.com/gitleaks/gitleaks
- 자경단 안전 정책: [`SAFETY.md`](./SAFETY.md)
- 사고 회고 보관: [`CHANGELOG.md`](./CHANGELOG.md)

---

**누출은 사냥감이다. 발견하면 즉시 잡고, 잡은 뒤엔 회고를 남긴다.**
