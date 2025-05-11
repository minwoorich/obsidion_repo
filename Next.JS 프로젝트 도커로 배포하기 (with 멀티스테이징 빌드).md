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

> 멀티스테이징 빌드 (Multi-stage build) 는 docker 에서 사용되는 빌드 기법으로, 하나의 Dockerfile 내에서 여러 단계의 빌드 과정을 정의하여 최종 이미지의 크기를 줄이고 보안을 강화하는 방법입니다. 


### Dockerfile
```shell
# 0. 베이스 이미지 설정
FROM node:20-alpine AS base

# 1. 의존성 설치 스테이지
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* .npmrc* ./

RUN \

    if [ -f yarn.lock ]; then yarn --frozen-lockfile; \

    elif [ -f package-lock.json ]; then npm ci; \

    elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i; \

    else echo "Lockfile not found." && exit 1; \

    fi

# 2. 빌드 스테이지
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED 1

RUN \

    if [ -f yarn.lock ]; then yarn build; \

    elif [ -f package-lock.json ]; then npm run build; \

    elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm run build; \

    else echo "Lockfile not found." && exit 1; \

    fi

# 3. 프로덕션 스테이지
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

파트별로 도커 파일 스크립트를 분석 해보도록 하겟다.