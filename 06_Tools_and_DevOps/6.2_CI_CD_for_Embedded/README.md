# 6.2 CI/CD for Embedded

## Topics Covered

### Jenkins / GitHub Actions / GitLab CI

#### Continuous Integration (CI)
- **Automated Builds**: Every commit triggers build
- **Automated Tests**: Unit tests, integration tests
- **Static Analysis**: Code quality checks
- **Artifacts**: Store build outputs

#### Continuous Deployment (CD)
- **Automated Deployment**: Push to staging/production
- **Over-The-Air (OTA)**: Firmware updates
- **Rollback**: Revert to previous version

### Jenkins

#### Overview
- **Self-Hosted**: Run on your own server
- **Plugins**: Extensive plugin ecosystem
- **Pipelines**: Jenkinsfile (Groovy DSL)

#### Jenkinsfile Example
```groovy
pipeline {
    agent any
    
    environment {
        TOOLCHAIN_PATH = '/opt/gcc-arm-none-eabi'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/user/embedded-project.git'
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    export PATH=$TOOLCHAIN_PATH/bin:$PATH
                    mkdir -p build
                    cd build
                    cmake ..
                    make -j$(nproc)
                '''
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    cd build
                    ctest --output-on-failure
                '''
            }
        }
        
        stage('Static Analysis') {
            steps {
                sh 'cppcheck --enable=all --xml src/ 2> cppcheck.xml'
                recordIssues(tools: [cppCheck(pattern: 'cppcheck.xml')])
            }
        }
        
        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'build/*.bin,build/*.hex',
                                 fingerprint: true
            }
        }
        
        stage('Deploy to Test Device') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    # Flash firmware to test device
                    openocd -f interface/stlink.cfg \
                            -f target/stm32f4x.cfg \
                            -c "program build/firmware.bin 0x08000000 verify reset exit"
                '''
            }
        }
    }
    
    post {
        success {
            emailext subject: "Build Successful: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                     body: "Build completed successfully.",
                     to: "team@example.com"
        }
        failure {
            emailext subject: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                     body: "Build failed. Please check console output.",
                     to: "team@example.com"
        }
    }
}
```

#### Jenkins Setup
```bash
# Install Jenkins (Ubuntu)
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt install jenkins

# Install required plugins
# - Git Plugin
# - Pipeline Plugin
# - Warnings Next Generation (for static analysis)
# - Email Extension Plugin
```

### GitHub Actions

#### Overview
- **Cloud-Hosted**: No infrastructure management
- **Free Tier**: 2000 minutes/month for private repos
- **YAML Configuration**: .github/workflows/

#### Workflow Example
```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      with:
        submodules: recursive
    
    - name: Install ARM toolchain
      run: |
        sudo apt-get update
        sudo apt-get install -y gcc-arm-none-eabi
    
    - name: Cache build artifacts
      uses: actions/cache@v3
      with:
        path: build
        key: ${{ runner.os }}-build-${{ hashFiles('**/CMakeLists.txt') }}
    
    - name: Build firmware
      run: |
        mkdir -p build
        cd build
        cmake ..
        make -j$(nproc)
    
    - name: Run unit tests
      run: |
        cd build
        ctest --output-on-failure
    
    - name: Run static analysis
      run: |
        sudo apt-get install -y cppcheck
        cppcheck --enable=all --error-exitcode=1 src/
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: firmware
        path: |
          build/*.bin
          build/*.hex
          build/*.elf
    
    - name: Create release
      if: startsWith(github.ref, 'refs/tags/')
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.ref }}
        release_name: Release ${{ github.ref }}
        draft: false
        prerelease: false

  code-quality:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run clang-format check
      run: |
        sudo apt-get install -y clang-format
        find src/ -name '*.c' -o -name '*.h' | xargs clang-format --dry-run --Werror
    
    - name: Check code size
      run: |
        cd build
        arm-none-eabi-size firmware.elf
        # Fail if code size exceeds limit
        CODE_SIZE=$(arm-none-eabi-size firmware.elf | awk 'NR==2 {print $1}')
        if [ $CODE_SIZE -gt 262144 ]; then
          echo "Code size $CODE_SIZE exceeds 256KB limit"
          exit 1
        fi
```

#### Matrix Builds
```yaml
strategy:
  matrix:
    platform: [stm32f4, stm32f7, nrf52]
    build_type: [Debug, Release]

steps:
- name: Build for ${{ matrix.platform }}
  run: |
    cmake -DPLATFORM=${{ matrix.platform }} \
          -DCMAKE_BUILD_TYPE=${{ matrix.build_type }} \
          .
    make
```

### GitLab CI

#### Overview
- **Integrated**: Built into GitLab
- **Runners**: Self-hosted or shared
- **YAML Configuration**: .gitlab-ci.yml

#### Pipeline Example
```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  GIT_SUBMODULE_STRATEGY: recursive

before_script:
  - export PATH=/opt/gcc-arm-none-eabi/bin:$PATH

build:firmware:
  stage: build
  image: ubuntu:22.04
  before_script:
    - apt-get update
    - apt-get install -y gcc-arm-none-eabi cmake make
  script:
    - mkdir -p build
    - cd build
    - cmake ..
    - make -j$(nproc)
  artifacts:
    paths:
      - build/*.bin
      - build/*.hex
      - build/*.elf
    expire_in: 1 week
  cache:
    paths:
      - build/

test:unit:
  stage: test
  image: ubuntu:22.04
  dependencies:
    - build:firmware
  script:
    - cd build
    - ctest --output-on-failure --verbose

test:static-analysis:
  stage: test
  image: ubuntu:22.04
  script:
    - apt-get update && apt-get install -y cppcheck
    - cppcheck --enable=all --xml src/ 2> cppcheck.xml
  artifacts:
    reports:
      codequality: cppcheck.xml

deploy:staging:
  stage: deploy
  only:
    - develop
  script:
    - echo "Deploying to staging environment"
    - ./scripts/flash_device.sh staging build/firmware.bin

deploy:production:
  stage: deploy
  only:
    - main
  when: manual
  script:
    - echo "Deploying to production"
    - ./scripts/flash_device.sh production build/firmware.bin
```

### Automated Hardware Testing (Lava, LabGrid)

#### Hardware-in-the-Loop (HIL) Testing

##### LabGrid
- **Purpose**: Automated testing with real hardware
- **Components**:
  - **Coordinator**: Manages resources
  - **Exporter**: Provides hardware resources
  - **Client**: Runs tests

##### Example Configuration
```yaml
# device.yaml
targets:
  main:
    resources:
      - RawSerialPort:
          port: /dev/ttyUSB0
          speed: 115200
      - NetworkPowerPort:
          model: netio4
          host: powerswitch.local
          index: 1
      - USBFlashableDevice:
          match:
            ID_SERIAL_SHORT: "ABC123"
    drivers:
      - SerialDriver: {}
      - PowerDriver:
          delay: 2.0
      - FlashDriver:
          image: firmware.bin
```

##### Test Script
```python
#!/usr/bin/env python3
import labgrid
from labgrid.protocol import PowerProtocol, ConsoleProtocol

# Get target
env = labgrid.Environment("device.yaml")
target = env.get_target("main")

# Power cycle device
power = target.get_driver(PowerProtocol)
power.cycle()

# Get console
console = target.get_driver(ConsoleProtocol)

# Wait for boot message
console.expect("System initialized", timeout=10)

# Send command
console.sendline("run_test")

# Check result
console.expect("Test passed", timeout=30)
```

#### LAVA (Linaro Automated Validation Architecture)
- **Purpose**: Large-scale automated testing
- **Use Case**: Embedded Linux testing
- **Features**: Job scheduling, test reporting, device management

##### LAVA Job Definition
```yaml
job_name: firmware-test
timeouts:
  job:
    minutes: 30
  action:
    minutes: 5
priority: medium

actions:
  - deploy:
      to: tftp
      kernel:
        url: http://server/zImage
      dtb:
        url: http://server/device.dtb
      rootfs:
        url: http://server/rootfs.tar.gz
  
  - boot:
      method: u-boot
      commands: nfs
      prompts:
        - 'root@device'
  
  - test:
      definitions:
        - repository:
            metadata:
              format: Lava-Test Test Definition 1.0
              name: basic-tests
            run:
              steps:
                - ./run_tests.sh
          from: inline
          name: basic-tests
          path: inline/basic-tests.yaml
```

### Docker for Build Environments

#### Reproducible Builds
```dockerfile
# Dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y \
    gcc-arm-none-eabi \
    cmake \
    make \
    git \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install additional tools
RUN pip3 install --no-cache-dir ceedling unity

# Set working directory
WORKDIR /workspace

# Copy project files
COPY . .

# Build command
CMD ["bash", "-c", "mkdir -p build && cd build && cmake .. && make"]
```

#### Build with Docker
```bash
# Build Docker image
docker build -t embedded-build:latest .

# Run build
docker run --rm -v $(pwd):/workspace embedded-build:latest

# Interactive shell
docker run --rm -it -v $(pwd):/workspace embedded-build:latest bash
```

#### Docker Compose for Complex Setup
```yaml
# docker-compose.yml
version: '3'

services:
  builder:
    image: embedded-build:latest
    volumes:
      - .:/workspace
    command: make

  tester:
    image: embedded-test:latest
    depends_on:
      - builder
    volumes:
      - .:/workspace
    command: ctest

  analyzer:
    image: cppcheck:latest
    volumes:
      - .:/workspace
    command: cppcheck --enable=all src/
```

## Key Concepts

- **Pipeline as Code**: Version control CI/CD configuration
- **Artifacts**: Store build outputs for deployment
- **Parallel Execution**: Speed up with parallel jobs
- **Caching**: Reuse dependencies between builds
- **Test Automation**: Reduce manual testing effort

## Practical Exercises

1. Set up Jenkins pipeline for embedded project
2. Create GitHub Actions workflow with matrix builds
3. Configure GitLab CI with multiple stages
4. Write automated test with LabGrid
5. Create Docker image for reproducible builds
6. Set up hardware-in-the-loop testing
7. Implement OTA firmware update in CD pipeline

## Best Practices

1. **Keep builds fast** (<10 minutes ideal)
2. **Fail fast** (run quick tests first)
3. **Use caching** to speed up builds
4. **Version control** build configurations
5. **Notify team** of build failures
6. **Test on real hardware** before release
7. **Use docker** for reproducible builds

## Metrics to Track

- **Build Success Rate**: % of successful builds
- **Build Time**: Time to build firmware
- **Test Coverage**: % of code covered by tests
- **Code Quality**: Static analysis issues
- **Deployment Frequency**: How often firmware is released

## Tools

- **Jenkins**: Self-hosted CI/CD
- **GitHub Actions**: Cloud-hosted, integrated with GitHub
- **GitLab CI**: Integrated with GitLab
- **LabGrid**: Hardware test automation
- **LAVA**: Large-scale embedded Linux testing
- **Docker**: Containerization for builds

## Resources

- Jenkins documentation
- GitHub Actions documentation
- GitLab CI/CD documentation
- LabGrid documentation
- Docker documentation
