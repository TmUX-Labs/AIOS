# AIOS (Artificial Intelligence OS)

AIOS는 x86_64 베어메탈 환경에서 인공지능 연산 가속을 실험하고 연구하기 위한 특수 목적 커널 프로토타입입니다. 

범용 운영체제(Linux, Windows 등)의 무거운 프로세스 스케줄러, 가상 파일 시스템(VFS) 레이어 등의 추상화 오버헤드를 배제하고, 하드웨어 제어권을 연산 엔진이 직접 통제하는 베어메탈 실행 환경 구축을 지향합니다.

---

## Key Architectural Specifications

### 1. Dedicated Compute Cores (인터럽트 격리 및 AP 코어 활용)

- 멀티코어(SMP) 환경 가동 시, 하드웨어 인터럽트 및 일반 I/O를 전담하는 부팅 코어와 연산 전담 코어를 논리적으로 분리합니다.

- 연산 코어의 비동기 인터럽트 처리를 차단함으로써, 태스크 스케줄링 및 컨텍스트 스위칭 오버헤드를 최소화하고 오직 텐서 루프 연산에만 클럭을 집중시킬 수 있는 구조를 연구합니다.

### 2. Huge Page Mapping & TLB Pressure Reduction

- 주소 변환 및 관리 구조를 직관적으로 유지하기 위해 가상 주소와 물리 주소를 1:1로 일치시키는 고정 매핑(Identity Mapping) 체계를 사용합니다. 

- 기본 4KB 페이지 대신 2MB 대형 페이지(Huge Page) 구조를 전면 기용합니다. 이는 동일한 수의 TLB 엔트리로 훨씬 넓은 가중치 메모리 공간을 커버할 수 있어, 주소 번역 오버헤드와 페이지 폴트 발생 가능성을 낮춥니다.

### 3. Tensor Arena (고정형 연속 범프 할당기)

- 가중치 로딩 및 추론(Inference) 과정에서 발생하는 텐서 데이터의 파편화를 방지하기 위해 일반적인 malloc이나 free 기능을 제공하지 않습니다. 

- 커널 내부에 거대한 연속 물리 메모리 블록인 Tensor Arena를 선점하고, 64바이트 경계 정렬 기반의 범프 할당기(Bump Allocator)를 구현하여 메모리 할당 오버헤드를 원천적으로 억제합니다.

---

## Development & Environment Status

- Target Hardware: x86_64 Architecture (Intel/AMD)

- Emulation: QEMU VM (-m 1G, -vga std, -display vnc=:1)

- Bootloader Spec: Multiboot Specification 규격 준수

- Paging Initialization: IA-32e 4단계 페이징(PML4, PDPT, PDT) 빌드 및 2MB Huge Page 기반 물리 메모리 1GB 고정 매핑 완료.

- Kernel Entry Setup: GDT(Global Descriptor Table) 수립 및 ljmp 파이프라인 플러시를 통한 64비트 롱 모드 진입, C 커널 본체(kernel_main) 바인딩 완료.

---

## Project Roadmap

- [x] Phase 1: Bootstrapping

  - Multiboot Header 수립 및 32비트 보호 모드 진입

  - 4단계 페이징을 위한 빌드 및 64비트 롱 모드 전환

  - C언어 커널 본체(kernel_main) 바인딩

- [x] Phase 2: Hardware Interrupt & I/O Subsystem

  - IDT(Interrupt Descriptor Table) 및 8259 PIC 초기화

  - PS/2 키보드 드라이버 및 로우 레벨 핸들러 구현

  - 인터럽트 안전 격리를 위한 Makefile 내 -mgeneral-regs-only 컴파일 플래그 분리 적용

- [x] Phase 3: Tensor Arena & Baseline Matrix Engine
  - 64바이트 정렬 기반 범프 할당기(Tensor Arena) 가동

  - CR0/CR4 레지스터 조작을 통한 CPU 내장 sse2 하드웨어 가속 유닛 잠금 해제

  - L1 캐시 적중률 최적화를 위한 i -> k -> j 루프 기반 행렬 곱셈 엔진 구현 및 VGA 덤프 검증

- [ ] Phase 4: Kernel Correctness Foundation(Current Step)

  - GRUB 부트로더가 넘겨준 EBX 주소 수거 및 Multiboot Memory Map 파싱 엔진 구현 (하드코딩된 RAM 매핑 제거)

  - 커널 안정성을 위한 하드웨어 예외 핸들러 적재

- [ ] Phase 5: Explicit SIMD & Implementation Dispatch

  - 컴파일러 자동 벡터화 최적화 분석 (-fopt-info-vec) 및 성능 측정기 도입

  - 명시적 SSE2 Intrinsic 커널 이식 및 CPUID 기반 기능 탐지(CPU Feature Manager) 구축

---

## How to Build and Run

```bash
# 기존 빌드 리소스 정리
$ make clean

# 부팅 및 커널 코드 컴파일, ISO 디스크 이미지 패키징 후 QEMU 실행
$ make run