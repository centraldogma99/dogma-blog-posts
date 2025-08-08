# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

이 저장소는 기술 블로그 포스트를 마크다운 형식으로 관리하는 프로젝트입니다. 주요 주제로는 React Router, Tanstack Query, Gemini AI, Electron 등을 다룹니다.

## 디렉토리 구조

루트 디렉토리에 블로그 포스트 마크다운 파일들이 위치합니다:
- `*.md` - 블로그 포스트 파일들

## 마크다운 포스트 구조

각 블로그 포스트는 다음과 같은 frontmatter 구조를 가집니다:
```yaml
---
title: 포스트 제목
subtitle: 부제목
date: YYYY-MM-DD
draft: true/false
tag:
  - tag1
  - tag2
---
```

## 작성 규칙

1. 모든 포스트는 한국어로 작성됩니다
2. 코드 블록에는 적절한 언어 표시를 포함합니다 (예: tsx, javascript)
3. 제목은 독자의 관심을 끌 수 있는 구체적인 내용을 담습니다
4. 실제 경험과 교훈을 바탕으로 실용적인 내용을 작성합니다

## 주요 포스트 카테고리

- **React & 웹 개발**: React Router, Tanstack Query 등 웹 프레임워크 관련
- **AI/LLM**: Gemini, 프롬프트 엔지니어링 등 AI 활용 사례
- **Desktop 앱**: Electron 등 데스크톱 애플리케이션 개발