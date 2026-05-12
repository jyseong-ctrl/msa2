# msa-ortholog

Ensembl REST API에서 사람(human) 유전자의 직교상동(ortholog) 단백질 서열을 가져오고, EBI Clustal Omega로 다중 서열 정렬(MSA)을 수행하는 TypeScript 라이브러리입니다.

Next.js 환경(App Router의 `fetch` revalidate 옵션 활용)에서 동작하도록 설계되었으며, 일반 Node.js 18+ 환경에서도 동작합니다.

## 주요 기능

- 사람 유전자 심볼 또는 Ensembl ID로부터 단백질 서열 조회
- Ensembl Compara를 이용한 직교상동 서열 가져오기 (기본 9종 + 사용자 지정 종)
- 신호 펩티드(signal peptide) 자동 감지 및 선택적 제거 (`include` / `trim` 모드)
- BioMart 기반 사람 유전자 인덱스 캐싱
- FASTA 포맷 생성 및 Clustal Omega 작업 제출/상태조회/결과 다운로드
- Clustal 결과 파싱 및 사람 서열 기준 동일성(% identity) 계산

## 설치

```bash
npm install
# 또는
pnpm install
# 또는
yarn install
```

## 빌드

```bash
npm run build       # dist/ 디렉터리에 컴파일
npm run typecheck   # 타입 검사만 수행
```

## 사용 예시

```ts
import {
  fetchOrthologs,
  submitClustalOmega,
  getClustalOmegaStatus,
  getClustalOmegaAlignment,
  parseClustal,
  summarizeIdentity,
  DEFAULT_TARGET_SPECIES
} from "./src/msa";

// 1) 직교상동 서열 가져오기
const result = await fetchOrthologs("TP53", DEFAULT_TARGET_SPECIES, {
  signalPeptideMode: "include" // 또는 "trim"
});

console.log(result.fasta);

// 2) Clustal Omega에 작업 제출
const jobId = await submitClustalOmega(result.fasta, "you@example.com");

// 3) 상태 폴링
let status = await getClustalOmegaStatus(jobId);
while (status === "PENDING" || status === "RUNNING") {
  await new Promise((r) => setTimeout(r, 5000));
  status = await getClustalOmegaStatus(jobId);
}

// 4) 정렬 결과 받기
if (status === "FINISHED") {
  const alignment = await getClustalOmegaAlignment(jobId);
  const aligned = parseClustal(alignment);
  const identities = summarizeIdentity(aligned, "human");
  console.log(identities);
}
```

## API 개요

| 함수 / 상수 | 설명 |
|---|---|
| `fetchOrthologs(query, species, opts)` | 사람 유전자에 대한 직교상동 단백질 서열을 가져와 FASTA로 반환 |
| `submitClustalOmega(fasta, email)` | EBI Clustal Omega REST에 작업 제출, jobId 반환 |
| `getClustalOmegaStatus(jobId)` | 작업 상태 조회 (`PENDING` / `RUNNING` / `FINISHED` / ...) |
| `getClustalOmegaAlignment(jobId)` | 정렬 결과(clustal_num 포맷) 텍스트 반환 |
| `parseClustal(text)` | Clustal 포맷 텍스트를 `{ id: sequence }` 객체로 파싱 |
| `summarizeIdentity(aligned, ref?)` | 참조 서열 대비 동일성(%) 계산 |
| `toFasta(records)` | `SequenceRecord[]`를 FASTA 문자열로 변환 |
| `DEFAULT_TARGET_SPECIES` | 기본 비교 대상 9종 (마우스, 랫, 침팬지 등) |
| `HUMAN_REFERENCE` | 사람 참조 종 정보 |
| `MAX_SELECTED_SPECIES_FOR_MSA` | MSA에서 권장하는 최대 종 개수 (80) |

## 데이터 소스

- [Ensembl REST API](https://rest.ensembl.org) — 유전자 / 단백질 / 직교상동 조회
- [Ensembl BioMart](https://www.ensembl.org/biomart/martservice) — 사람 유전자 인덱스
- [Ensembl Beta Gene Search](https://beta.ensembl.org/api/search/genes) — 유전자 검색
- [EBI Clustal Omega REST](https://www.ebi.ac.uk/Tools/services/rest/clustalo) — 다중 서열 정렬

각 서비스의 이용약관 및 호출 제한을 준수해 주세요. Clustal Omega 제출 시에는 유효한 이메일 주소 사용을 권장합니다.

## 요구 사항

- Node.js 18 이상 (전역 `fetch`, `AbortController` 필요)
- TypeScript 5.x
- (선택) Next.js 13 이상 — `fetch`의 `next.revalidate` 캐싱 활용 시

## 라이선스

MIT License — 자세한 내용은 [LICENSE](./LICENSE) 파일을 참고하세요.

## 기여

이슈와 PR을 환영합니다. 큰 변경은 먼저 이슈로 논의해 주세요.
