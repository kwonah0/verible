# Verible 아키텍처 정리

## 📌 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **목적** | SystemVerilog (IEEE 1800-2017) 파서 및 개발 도구 |
| **언어** | C++17 |
| **빌드** | Bazel |
| **라이선스** | Apache 2.0 |

### 주요 도구

| 도구 | 설명 |
|------|------|
| `verible-verilog-lint` | 스타일 린터 |
| `verible-verilog-format` | 코드 포매터 |
| `verible-verilog-syntax` | 구문 검사기 |
| `verible-verilog-ls` | LSP 언어 서버 |

---

## 🏗️ 전체 아키텍처

```
소스코드 → Lexer → 토큰 스트림 → Filter → Contextualizer → Parser → CST
              (Flex)                                        (Bison)
```

### 핵심 설계: 렉서/파서 분리 (Decoupled Design)

**전통적 방식:**
```
yyparse() ──직접 호출──▶ yylex()
```

**Verible 방식:**
```
Lexer ──▶ 토큰 스트림 ──▶ Filter ──▶ Contextualizer ──▶ Parser
  │                                        │               │
  │         (중간 변환 가능!)                │               │
  └────────────────────────────────────────┴───────────────┘
```

**장점:**
- 도구별 커스터마이징 가능
- 문맥 기반 토큰 변환 가능
- Unpreprocessed 코드 지원
- 각 단계 독립적 테스트 가능

---

## 🔧 파이프라인 상세

### 1단계: Lexer (Flex)

```
"module foo;" → [TK_module] [SymbolIdentifier "foo"] [';']
```

- **도구**: Flex (1987~, Lex의 오픈소스 버전)
- **입력**: `verilog.lex` (정규표현식 규칙)
- **출력**: 토큰 스트림

```lex
// verilog.lex 예시
"module"    { return TK_module; }
"always_ff" { return TK_always_ff; }
[a-zA-Z_][a-zA-Z0-9_$]*  { return SymbolIdentifier; }
```

### 2단계: Filter

- 주석, 공백 제거 (파싱용)
- 원본 토큰은 보존 (포매터용)

### 3단계: Contextualizer

**역할**: 문맥에 따라 토큰 의미 변환 (LR 파서 한계 극복)

| 토큰 | 문맥 1 | 문맥 2 |
|------|--------|--------|
| `->` | constraint implication | event trigger |
| `with` | randomize call | array method |
| `:` | label | range |

### 4단계: Parser (Bison)

- **도구**: Bison (1985~, Yacc의 GNU 버전)
- **알고리즘**: LALR(1)
- **입력**: `verilog.y` (BNF 문법 규칙)
- **출력**: CST (Concrete Syntax Tree)

```yacc
// verilog.y 예시
module_declaration
  : module_header module_item_list_opt TK_endmodule
    { $$ = MakeTaggedNode(N::kModuleDeclaration, $1, $2, $3); }
  ;
```

---

## 🌳 CST (Concrete Syntax Tree)

### AST vs CST

| AST | CST |
|-----|-----|
| 의미적 정보만 보존 | **모든 토큰 보존** |
| 구두점, 공백 제거 | 구두점, 세미콜론 포함 |
| 원본 복원 불가 | **원본 완벽 복원 가능** |
| 컴파일러용 | **린터/포매터 필수** |

### 클래스 구조

```
Symbol (추상 기본 클래스)
    │
    ├── SyntaxTreeLeaf (터미널 - 토큰)
    │       └── TokenInfo token_
    │
    └── SyntaxTreeNode (비터미널 - 노드)
            ├── int tag_ (NodeEnum)
            └── vector<SymbolPtr> children_
```

### CST 예시

**입력:**
```verilog
module foo;
  assign x = 1;
endmodule
```

**CST:**
```
kModuleDeclaration
├── kModuleHeader
│   ├── [Leaf] TK_module "module"
│   ├── [Leaf] SymbolIdentifier "foo"
│   └── [Leaf] ';' ";"
├── kModuleItemList
│   └── kContinuousAssignmentStatement
│       ├── [Leaf] TK_assign "assign"
│       └── ...
└── kEnd
    └── [Leaf] TK_endmodule "endmodule"
```

### NodeEnum (노드 열거형)

- **위치**: `verible/verilog/CST/verilog-nonterminals.h`
- **개수**: 475개 이상
- **⚠️ 불안정**: 문법 변경 시 바뀔 수 있음

### 토큰 vs 노드 열거형 안정성

| 구분 | 안정성 | 이유 |
|------|--------|------|
| **토큰 열거형** | ✅ 안정적 | 언어 표준에 고정 |
| **노드 열거형** | ⚠️ 불안정 | 문법 리팩토링 시 변경 가능 |

### Accessor 함수 사용 권장

```cpp
// ❌ 직접 접근 (위험)
const auto& name = node[0][2];

// ✅ Accessor 사용 (안전)
const auto* name = GetModuleName(module_decl);
```

**이유:**
- 내부 구조 변경에 대한 방어
- 코드 의도 명확
- 단위 테스트 용이

---

## 📊 표준 지원 수준

### 목표 및 현황

| 항목 | 내용 |
|------|------|
| **목표 표준** | IEEE 1800-2017 |
| **현재 상태** | "vast majority" (대부분) 지원 |
| **실무 커버리지** | ~95%+ |
| **검증** | sv-tests 정기 테스트 |

### 규모

| 항목 | 개수 |
|------|------|
| 문법 규칙 (verilog.y) | 8,712 줄 |
| 렉서 규칙 (verilog.lex) | 1,668 줄 |
| 노드 타입 | 475개 |
| 토큰 타입 | 467개 |

### 지원 기능

**✅ 완전 지원:**
- module, interface, program, class
- function, task
- always, always_ff, always_comb, always_latch
- generate 구문
- 데이터 타입 (logic, reg, wire, struct, union, enum)
- 매크로/전처리 지시자
- assertions (SVA), constraints, covergroup

**⚠️ 제한:**
- 일부 복잡한 전처리 패턴
- 일부 에지 케이스 문법

---

## ⚠️ 파싱 실패 시 동작

### 처리 흐름

```
파싱 실패 발생
    │
    ├── 1. 에러 메시지 생성
    ├── 2. RejectedToken 저장
    ├── 3. 자동 재시도 (다른 파싱 모드)
    ├── 4. 부분 CST 생성 (가능한 경우)
    └── 5. 분석 계속 진행 (옵션)
```

### 에러 정보 구조

```cpp
struct RejectedToken {
  TokenInfo token_info;      // 문제의 토큰
  AnalysisPhase phase;       // 렉싱/전처리/파싱
  std::string explanation;   // 에러 설명
  ErrorSeverity severity;    // 에러/경고
};
```

### 도구별 동작

| 도구 | 파싱 실패 시 |
|------|-------------|
| **린터** | 에러 출력 + 부분 CST로 검사 계속 |
| **포매터** | 에러 출력 + 원본 유지 또는 부분 포매팅 |
| **구문 검사기** | 에러 출력 + 부분 트리 출력 가능 |

### 에러 메시지 예시

```
test.sv:15:23: syntax error, unexpected identifier [syntax]
  assign x = y z;
                ^
```

---

## 🛠️ 문법 수정 방법

### 난이도별 정리

| 작업 | 난이도 | 수정 파일 |
|------|--------|----------|
| 새 키워드 추가 | ⭐ 쉬움 | verilog.lex + verilog.y |
| 새 연산자 추가 | ⭐ 쉬움 | verilog.lex + verilog.y |
| 새 구문 추가 | ⭐⭐ 보통 | + verilog-nonterminals.h |
| 기존 문법 수정 | ⭐⭐⭐ 어려움 | + Accessor + 테스트 |
| 충돌 해결 | ⭐⭐⭐⭐ | Bison 내부 이해 필요 |

### 추가 예시

```lex
// verilog.lex - 키워드 추가
"my_keyword" { UpdateLocation(); return TK_my_keyword; }
```

```yacc
// verilog.y - 문법 규칙 추가
%token TK_my_keyword

my_statement
  : TK_my_keyword expression ';'
    { $$ = MakeTaggedNode(N::kMyStatement, $1, $2, $3); }
  ;
```

---

## 📁 주요 파일 경로

| 파일 | 경로 |
|------|------|
| 렉서 문법 | `verible/verilog/parser/verilog.lex` |
| 파서 문법 | `verible/verilog/parser/verilog.y` |
| 노드 열거형 | `verible/verilog/CST/verilog-nonterminals.h` |
| Contextualizer | `verible/verilog/parser/verilog-lexical-context.h` |
| CST Accessor | `verible/verilog/CST/*.h` |
| 분석기 | `verible/verilog/analysis/verilog-analyzer.cc` |

---

## 🔗 참고 자료

- [Verible GitHub](https://github.com/chipsalliance/verible)
- [sv-tests (표준 준수 테스트)](https://github.com/chipsalliance/sv-tests)
- [IEEE 1800-2017 (SystemVerilog LRM)](https://ieeexplore.ieee.org/document/8299595)
- [Flex 공식 문서](https://www.gnu.org/software/flex/)
- [Bison 공식 문서](https://www.gnu.org/software/bison/)
