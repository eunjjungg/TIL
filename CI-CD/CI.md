## CI: Continous Integration

CI를 위한 두 가지 조건

- 코드 변경사항을 main branch에 개발자가 주기적으로 merge해야됨. 왜냐하면 오랜 시간동안 merge하지 않다가 올리게 되면 conflict가 발생하므로 해결하는데에도 오랜 시간이 소요됨.
- 통합을 위한 단계의 자동화가 되어야 함. 주로 CI script를 구현해서 이 스크립트가 실행해줌. 스크립트 통과가 되면 초록이 안 되면 빨강이 나오게 됨.

CI의 장점? : merge conflict를 피할 수 있고 생기더라도 작은 단위의 문제가 발생하기 때문에 발견이 빠르고 작은 단위 기준 해결이 가능함. 대규모 협업 시 큰 이득이 있음. 

<br/>

### Github Actions

- CI/CD tool 중 하나임.
- yaml(야믈) 언어를 사용하는 .yml 파일의 스크립트를 읽어 CI/CD를 수행함.

<br/>

### 용어

**workflow**

- 최상위 개념으로 자동화된 전체 프로세스를 말함
- 여러 Job으로 구성됨
- Event에 의해 예약되거나 트리거 됨
- github actions에서는 .github/workflows/_.yml로 저장됨

**event**

- Workflow를 Trigger하는 활동과 규칙
- 다음과 같은 이벤트가 있음
    - 특정 브랜치로의 push, pr
    - 정해진 시간대에 반복

**job**

- VM 환경에서 실행됨
- 여러 step들로 이루어져 있으며 병렬 실행 가능

**step**

- job에서 실행할 프로세스 단위
- steps 내에서 `-`로 각 프로세스들을 구분
- uses : 해당 스텝에서 수행할 액션
- name : 해당 스텝 이름
- run : 해당 step에서 실행될 커맨드 라인
- with : 해당 step에서 수행되는 Action에 정의되는 param. key-value 형태

**action**

- workflow 안에서 가장 작은 단위
- 개인이 만든 action을 사용할수도 있고 Marketplace의 공용 action을 사용할 수도 있음

**runner**

- workflow가 실행될 인스턴스

```yaml
# yml file name
name: Android CI

# event 발생 
on:
  # <main, dev, feat> branch로 pr 올리면 아래 jobs 수행
  pull_request:
    branches:
      - 'master'
      - 'develop'
      - 'feat/*'

jobs:
  build:

		# 우분투 환경에서 최신 버전으로 실행됨 (VM) 
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
      - name: set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
          cache: gradle

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build with Gradle
        run: ./gradlew build

```

- main, dev, feat branch로 pr 올리면 아래 jobs 수행
- uses: actions/checkout@v3 → 소스 코드를 가져옴
- uses: actions/setup-java@v3 → JDK 설정

**특수 케이스**

- case1 : A branch의 코드를 수정해서 A’가 되었다고 가정. 이때 A branch’ Github actions yml ≠ A’ branch’ Github actions yml이고 PR은 merge n commits into A from A’로 올렸다고 가정. 이때 Github Actions가 동작하는 yml 파일은 A’의 github actions로 동작함.

<br/>

### [Local.properties](http://Local.properties) 사용 시

vcs에 local.properties를 추가하지 않는다면 오류가 발생하게 됨. 따라서 Github repo setting에서 Secrets and variables > Actions > New repository secret을 눌러 secret 생성해주면 됨. 

![image](https://github.com/eunjjungg/TIL/assets/100047095/74950577-82b0-4c46-8851-b1afa613a40c)

생성해줬으면 아래와 같이 스크립트 내부에 사용해주면 됨

```yaml
- name: Create secret key
	run: |
		echo TEST_KEY=\"$API_KEY\" >> ./local.properties
		echo TEST_KEY2=\"$API_KEY2\" >> ./local.properties
	shell: bash
	env:
		API_KEY: ${{ secrets.TEST_KEY }}
		API_KEY2: ${{ secrets.TEST_KEY2 }}

```

<br/>

### ktlint check

ci를 할 때 ktlint도 체크해줄 수 있음. 단 project에 dependency를 추가해주어야 함. ktlint dependency 추가는 project 단의 build.gradle 파일에 추가해주어야 함. groovy - build.gradle의 경우 아래와 같이 작성하면 됨. 

```groovy
buildscript {
    repositories {
        maven {
            url 'https://plugins.gradle.org/m2/'
        }
    }
    dependencies {
        classpath "org.jlleitschuh.gradle:ktlint-gradle:9.1.0"
    }
}

plugins {
    id 'org.jlleitschuh.gradle.ktlint' version '9.1.0'
}
```

이후 yml 파일에 아래와 같이 추가해주면 됨.

```yaml
# steps 하위에 작성
# ktlint test
- name: Run ktlint
	run: ./gradlew ktlintCheck
```

<br/>

### gradle 캐싱

```yaml
- name: Cache Gradle packages
  uses: actions/cache@v2
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
  key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties', '**/buildSrc/**/*.kt') }}
  restore-keys: |
    ${{ runner.os }}-gradle-
```

위와 같이 캐싱을 하면 아래 사진의 주황색 밑줄 친 작업 두 개가 추가됨. 

![image](https://github.com/eunjjungg/TIL/assets/100047095/66aed196-3ece-4010-a9b3-6596e1a8d9f8)
### 최종본

local.properties의 작업은 아직 추가하지 않아서 최종본은 아래와 같음. 

```yaml
name: Android CI

on:
  push:
    branches:
      - 'hotfix/*'
  # main, dev, feat branch pr 올리면 아래 jobs 수행
  pull_request:
    branches:
      - 'master'
      - 'develop'
      - 'feat/*'

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
      # timestamp
      - name: Print start time
        run: |
          echo "Workflow started at $(date)"

      - uses: actions/checkout@v3
      - name: set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
          cache: gradle

      # gradle 캐싱 작업
      - name: Cache Gradle packages
        uses: actions/cache@v2
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties', '**/buildSrc/**/*.kt') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build with Gradle
        run: ./gradlew build

      # ktlint test
      - name: Run ktlint
        run: ./gradlew ktlintCheck
```

### Reference

- [https://heegs.tistory.com/92](https://heegs.tistory.com/92)
- [https://zzsza.github.io/development/2020/06/06/github-action/](https://zzsza.github.io/development/2020/06/06/github-action/)
- [https://blog.benelog.net/ktlint.html#gradle_빌드_설정](https://blog.benelog.net/ktlint.html#gradle_%EB%B9%8C%EB%93%9C_%EC%84%A4%EC%A0%95)
- [https://sungbin.land/github-actions-으로-안드로이드-ci-cd-구축하기-1aaaa6595c4a](https://sungbin.land/github-actions-%EC%9C%BC%EB%A1%9C-%EC%95%88%EB%93%9C%EB%A1%9C%EC%9D%B4%EB%93%9C-ci-cd-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0-1aaaa6595c4a)
