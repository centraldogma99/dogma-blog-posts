---
title: 'Electron Mac 앱 빌드 시 "손상된 파일" 오류 해결하기'
subtitle: 'ad-hoc 코드 서명'
date: 2025-08-01
draft: false
tag:
  - electron
  - javascript
---
## 문제 상황
Electron으로 Mac OS 앱을 빌드한 후, 다른 사용자에게 배포했을 때 다음과 같은 오류가 발생했습니다:

> "손상되었기 때문에 열 수 없습니다. 해당 항목을 휴지통으로 이동해야 합니다"

흥미롭게도 개발했던 기기에서 빌드 후 설치했을 때는 정상적으로 작동했지만, 빌드한 `.dmg` 파일을 slack이나 google drive를 통해 다른 기기에 전달하여 설치하려고 하니 위와 같은 오류가 발생하며 실행되지 않았습니다.

## 원인

이 문제는 **코드 서명(Code Signing)**이 없어서 발생하는 것으로, macOS의 Gatekeeper 보안 기능이 서명되지 않은 앱을 차단하기 때문입니다.

## 해결 방법

### 방법 1: 사용자가 직접 보안 우회

사용자가 터미널에서 다음 명령어를 실행하여 보안 속성을 제거할 수 있습니다:

```bash
xattr -r -d com.apple.quarantine /path/to/name.app
```

하지만 이 방법은 사용자에게 추가적인 작업을 요구하므로 사용자 경험 측면에서 좋지 않습니다.

### 방법 2: Ad-hoc 서명 (권장)

Electron Builder를 사용하는 경우, `package.json`의 `mac` 필드에 다음 설정을 추가합니다:

```json
{
  "build": {
    "mac": {
      "identity": null,
      "type": "development"
    }
  }
}
```

이 설정을 추가하면 ad-hoc 방식으로 서명되어, 다른 사람에게 넘겨주더라도 추가 작업 없이 앱을 실행할 수 있습니다.

