## 0. 서론
최근 회사에서 NextJS 프로젝트를 서버에 배포하는 임무를 맡아 진행하게 되었다. 스프링은 몇 번 배포해봤지만 Next JS 는 아직 배포 해본 적이 없어서 인터넷과 ai 를 적극 활용하였다. 사실 스프링 어플리케이션 도커 배포와 크게 다를 건 없었지만 그래도 처음은 처음인 지라 (사실 NextJS 자체가 처음,,) 조금 시행 착오를 겪었다. 그래서 간만에 블로그 글도 채울 겸 내가 겪었던 시행착오 및 새로 알게 된 것들을 기록해보고자 한다. 

## 1. 문제점: 너무 무거운 Docker 이미지

### Dockerfile
```shell
FROM node:20-alpine

WORKDIR /app

# 의존성 파일 복사
COPY package.json package-lock.json* ./

# 의존성 설치
RUN npm ci

# 소스 코드 복사
COPY . .
  
# Next.js 애플리케이션 빌드
RUN npm run build
  
# 애플리케이션 실행을 위한 포트 노출
EXPOSE 3000

# 애플리케이션 실행
CMD ["npm", "start"]
```
사실 조금 급히 배포를 했어야하는 상황이라 인터넷을 최대한 빨리 검색하여 급히 Dockerfile 을 작성하였는데 그러다보니 도커이미지 용량이 너무 무거워지는 일이 발생했다. 

![[Pasted image 20250511162629.png]]
보다시피 1.34 GB 를 차지하고 있는 것을 확인 할 수 있다.  같은 서버 내에 스프링 부트 이미지는 250MB 밖에 되지 않는 걸 생각해본다면 너무 말도 안 되게 무거운 상태였다. 게다가 현재 배포한 서버가 개발 서버도 아닌 임시 방편으로 잠깐 얻은 서버라서 용량이 간당간당한 상태인지라 용량관리를 잘 해줘야하는 상태였다.

## 2. 멀티 스테이징 빌드를 통한 이미지 용량 개선

그래서 이후 공식문서를 찾아본 결과 사실상 정답지에 가까운 Dockerfile 예시를 얻을 수 있었다.
[Vercel/nextJs 레포지토리](https://github.com/vercel/next.js/blob/canary/examples/with-docker/Dockerfile)
하지만 바로 Dockerfile 을 보기 전에 **멀티 스테이징 빌드** 가 무슨 말인지 잠깐 설명을 하고 넘어가보도록 하자.

### 멀티 스테이징 빌드 란?

> 멀티스테이징 빌드 (Multi-stage build) 는 Docker 에서 사용되는 빌드 기법으로, 하나의 Dockerfile 내에서 여러 단계의 빌드 과정을 정의하여 최종 이미지의 크기를 줄이고 보안을 강화하는 방법입니다.
> **_from Claude_** 

Dockerfile 을 작성 할 때 기본적으로는 빌드에 필요한 기본 이미지를  ``FROM`` 구문을 이용하여 지정을 해주고, 그 위에 필요한 환경 설정, 패키지 설치, 소스 코드 복사, 컴파일 등의 작업을 수행하게 된다.

하지만 이 경우 몇가지 문제점들이 발생하는데
1. 빌드에 필요한 도구 및 의존성 들이 최종 이미지에 모두 포함되어 이미지 크기가 커진다
2. 운영환경에서는 불필요한 빌드 툴이 포함되므로 보안 취약점이 증가할 수 있다

등이 존재한다.

이러한 문제점들을 해결하고자 Docker 17.05 버전 이후부터 **멀티 스테이징 빌드** 라는 고급 기능이 도입 되었다.

어떻게 더 문제점들을 개선했는지는 아래 실제 Dockerfile 과 함께 같이 살펴보자.

```shell
# 0. 베이스 이미지 설정
FROM node:20-alpine AS base

# 1. 빌드 스테이지 (의존성 설치 + 빌드 통합)
FROM base AS builder
RUN apk add --no-cache libc6-compat
WORKDIR /app

# 의존성 파일 복사 및 설치
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* .npmrc* ./
RUN \

    if [ -f yarn.lock ]; then yarn --frozen-lockfile; \

    elif [ -f package-lock.json ]; then npm ci; \

    elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i; \

    else echo "Lockfile not found." && exit 1; \

    fi
 
# 소스 코드 복사 및 빌드
COPY . .
ENV NEXT_TELEMETRY_DISABLED 1
RUN \

    if [ -f yarn.lock ]; then yarn build; \

    elif [ -f package-lock.json ]; then npm run build; \

    elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm run build; \

    else echo "Lockfile not found." && exit 1; \

    fi

  
# 2. 실행 스테이지
FROM base AS runner
WORKDIR /app
ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME=0.0.0.0

CMD ["node", "server.js"]
```

꽤 기니깐 차근차근 파트별로 나눠서 보자면 우선 

위 Dockerfile 은  **빌드**, **실행** 스테이지로 크게 **두 가지** 로 구분이 된다. (베이스 이미지 설정 하는 부분 제외)

### 빌드 스테이지
```shell
# 1. 빌드 스테이지 (의존성 설치 + 빌드 통합)
FROM base AS builder
RUN apk add --no-cache libc6-compat
WORKDIR /app

# 의존성 파일 복사 및 설치
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* .npmrc* ./
RUN \

    if [ -f yarn.lock ]; then yarn --frozen-lockfile; \

    elif [ -f package-lock.json ]; then npm ci; \

    elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i; \

    else echo "Lockfile not found." && exit 1; \

    fi
 
# 소스 코드 복사 및 빌드
COPY . .
ENV NEXT_TELEMETRY_DISABLED 1
RUN \

    if [ -f yarn.lock ]; then yarn build; \

    elif [ -f package-lock.json ]; then npm run build; \

    elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm run build; \

    else echo "Lockfile not found." && exit 1; \

    fi
```

우선 해당 스테이지는 빌드에 필요한 의존성들을 모두 다운 및 설치 받는 스테이지며 한 줄 한 줄 분석해보자.

**``FROM base AS builder ``**
base 이미지로 부터 builder 라는 스테이지를 생성

**``RUN apk add --no-cache libc6-compat ``**
Alpine Linux 의 경우 경량화를 위해 최소한으로만 설치된 리눅스이다. 그렇기 때문에 해당 이미지 기반에서 특정 프로그램 실행시 호환이 안되는 문제가 발생하는 경우가 발생한다. 그래서 혹시나 이런 문제를 방지하고자  libc6-compat 호환성 라이브러리를 설치하는것이다.

**``WORKDIR /app``**
작업 디렉토리 영역을 ``/app`` 으로 지정

**``COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* .npmrc* ./``**
현재 프로젝트 루트에 위치해있는 **패키지 관리 파일** 들을 Docker 의 작업 디렉토리(``/app``)로 복사하는것. 패키지 관리자가 각각 다를 경우를 대비해 모든 패키지 관리 파일들을 나열해놓았다. 

**RUN if ....**
해당하는 패키지 관리자에 맞게 의존성을 설치하는 명령어. 이때, yarn 의 ``--frozen-lockfile`` 혹은 ``npm ci`` 와 같이 오직 lock 파일에 명시된 버전만 설치 되도록(frozen 모드) 옵션을 걸어두었다. (pnpm 은 기본적으로 frozen 모드로 동작)

**``COPY . .``**
현재 프로젝트의 모든 소스코드 및 파일 복사

**``ENV NEXT_TELEMETRY_DISABLED 1``**
Next JS 는 



> 🤔 알았어,,, 근데 왜 굳이 의존성 설치 스테이지로 따로 나눠놓았을까?

우리가 코드를 수정하고, 통합하고, 배포하는 과정에서 과연 의존성 파일이 수정 되는 경우가 얼마나 될까?
대부분의 수정은 코드 변경이 주를 이룬다. 즉, 의존성 파일은 최초 배포 이후에는 자주 변경되지 않게 된다. 그렇다면 코드 수정 이후 **매번 빌드할 때마다 의존성을 새로 다운받고 설치하는것은 매우 비효율적**이고 빌드 시간을 많이 잡아먹게 될 것이다. 

하지만 스테이지를 나눌 경우 의존성 설치 스테이지(deps) 는 한번 빌드 이후에는 캐싱이 되어 다음 빌드 때는 **캐싱된 레이어를 그대로 재사용** 할 수 있게 된다.  즉, 소스 코드가 변경되어 다시 빌드를 하더라도 의존성 설치 부분은 기존에 사용한 것을 재활용하므로 **빌드시간을 대폭 단축** 시킬 수 있다. 

> 🤔 근데 도커 이미지는 원래 레이어 별로 캐싱 되는거 아니야? 굳이 멀티스테이징을 해야해?

자 그럼 처음에 작성했던 Dockerfile 을 살펴보자. 
```shell
FROM node:20-alpine
WORKDIR /app
COPY package.json ./          # 레이어 1
RUN npm install               # 레이어 2
COPY . .                      # 레이어 3 (소스 코드가 변경되면 이 레이어부터 캐시 무효화)
RUN npm run build             # 레이어 4 (소스 코드가 변경되면 항상 다시 실행됨)
```
원래 Dockerfile 의 각 명령어는 하나의 레이어를 생성하기 때문에 위와 같이 단일 스테이지에서도 레이어 캐싱은 동작을 한다. 하지만 문제는 