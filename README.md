# CMAgent Sentence-level Evaluation Workflow

## 1. 평가 목적

CMAgent의 추출 모델은 문장을 입력받아 해당 문장에 나타난 cell-marker 관계를 추출하고 표준화한다.
따라서 평가는 문장별 정답 관계를 기준으로 수행해야 한다.

현재 CellMarker 데이터와 CMAgent 모델 사이에는 데이터 단위 차이가 있다.

```text
CellMarker: 문서 단위 정답
  "이 PMID의 논문에 이 cell-marker 관계가 보고되었다."

CMAgent: 문장 단위 예측
  "이 문장에서 이 cell-marker 관계가 추출되었다."
```

CellMarker의 관계를 모델 예측과 바로 비교하면 해당 관계가 실제 평가 문장에 있는지 알 수 없다.
따라서 동일 PMID의 후보 문장에서 CellMarker 관계가 실제로 언급된 문장을 먼저 역추적해야 한다.

이 작업의 최종 목표는 다음과 같은 sentence-level gold test set을 만드는 것이다.

```csv
row_id,PMID,sentence,gold_cell,gold_marker,relation_label
969,28576768,"Myoepithelial cells expressed Cnn1 and Actg2.",Myoepithelial cell,Cnn1,POS
969,28576768,"Myoepithelial cells expressed Cnn1 and Actg2.",Myoepithelial cell,Actg2,POS
```

## 2. 사용하는 데이터

### 문장 후보

`data/processed/refined_positive_sentence_candidates_4588.csv`

- LLM으로 양성 후보를 정제한 4,588개 문장이다.
- 570개의 고유 PMID를 포함한다.
- 추출 모델과 표준화 파이프라인의 입력 데이터다.

### 문서 단위 정답

`data/raw/Cell_marker_db/all_cell_marker.txt`

- 2,537,570개 행과 10,940개의 고유 PMID를 포함한다.
- 평가에 필요한 주요 필드는 `pmid`, `cell_name`, `cell_name_class`, `marker`, `symbol`, `gene_name`이다.
- `single_cell_marker.txt`의 모든 고유 행과 `(PMID, cell_name, symbol)` 관계를 포함한다.
- 따라서 두 파일을 별도로 병합하지 않고 `all_cell_marker.txt`를 통합 정답 원본으로 사용한다.

`single_cell_marker.txt`의 고유 전체 행은 416,681개이며, 이 행들은 모두 `all_cell_marker.txt`에 존재한다.

### 이미 생성된 모델 결과

`results/full/mapped/prompt3_ontology_mapped.csv`

- 모델이 추출한 cell-marker 관계와 ontology/ID 표준화 결과다.
- gold 후보를 찾는 기준으로 사용하지 않는다.
- sentence-level gold가 확정된 후에만 비교 대상으로 사용한다.

모델 예측을 gold 생성에 직접 사용하지 않는 이유는 순환 평가를 방지하기 위해서다.
다만 GPT/Opus 엔터티 필드는 자동 매칭의 애매함을 표시하는 보조 신호로만 사용한다.

## 3. 전체 처리 흐름

```text
all_cell_marker.txt
문서 단위 정답 (PMID, cell, marker)
        |
        | PMID가 같은 관계만 선택
        v
refined_positive_sentence_candidates_4588.csv
4,588개 후보 문장
        |
        | 문장 안의 cell + marker 동시 등장 여부 검색
        v
sentence-level gold 후보
        |
        | HIGH_CONFIDENCE / NEEDS_REVIEW / REJECTED 자동 분류
        v
수동 검수
        |
        | 승인 관계를 한 행에 한 pair로 변환
        v
최종 sentence-level gold test set
        |
        | 기존 추출·표준화 결과와 비교
        v
Precision / Recall / F1
```

## 4. PMID 기준 후보 축소

먼저 후보 문장의 PMID가 `all_cell_marker.txt`에 존재하는지 확인한다.

```text
전체 후보 문장                         4,588개
후보 문장의 고유 PMID                    570개
CellMarker와 겹치는 PMID                 165개
겹치는 PMID에 속한 후보 문장           1,680개
CellMarker로 평가할 수 없는 후보 문장  2,908개
```

겹치는 165개 PMID에는 다음 규모의 CellMarker 관계가 있다.

```text
CellMarker 원본 행                    65,939개
고유 (PMID, cell_name, symbol) 관계  39,381개
```

따라서 현재 gold test set은 4,588개 전체가 아니라 1,680개 문장을 대상으로 구축한다.
나머지 2,908개를 NEG로 판단해서는 안 된다. CellMarker에 PMID가 없어 정답 여부를 판정할 수 없는 문장이므로 이번 평가에서 제외한다.

## 5. 문장 안에서 관계 찾기

각 후보 문장에 대해 동일 PMID의 CellMarker 행만 불러온다.

예를 들어 후보 문장이 다음과 같다고 가정한다.

```text
PMID: 12345
Sentence: CD8 T cells strongly expressed PDCD1.
```

동일 PMID의 CellMarker에 다음 행이 있다면 sentence-level 관계 후보가 된다.

```text
PMID: 12345
cell_name: CD8+ T cell
symbol: PDCD1
```

검색 시 다음 표현을 사용한다.

- Marker: `symbol`, `marker`, `gene_name` 순서로 검색한다.
- Cell: `cell_name` 정확 일치, `+`·공백·복수형 정규화, 제한적인 약어, `cell_name_class` 순서로 검색한다.
- 반드시 동일한 PMID 안에서 검색한다.
- 반드시 같은 CellMarker 행의 cell과 marker가 함께 잡혔는지 확인한다.

`CD8+ T cell`과 `CD8 + T cells`처럼 단순 표기 차이는 정규화할 수 있다.
반면 `Regulatory T cell`과 `Treg`, 세부 subtype과 상위 cell class의 차이는 오매칭 위험이 있어 자동 확정하지 않는다.

## 6. 자동 분류 결과

자동 매칭 결과는 `data/processed/cellmarker_sentence_gold_candidates.csv`에 저장한다.

```text
HIGH_CONFIDENCE      25개
NEEDS_REVIEW        848개
REJECTED            807개
전체               1,680개
```

이 상태값은 최종 정답이 아니라 수동 검수 우선순위를 정하기 위한 임시 분류다.

### HIGH_CONFIDENCE

다음 조건을 모두 만족하는 문장이다.

```text
동일 PMID
+ cell_name 정확/정규화 일치
+ marker 또는 symbol 일치
+ 동일 CellMarker 행의 cell-marker 쌍
+ 양성 관계 표현 존재
+ 후보 관계가 하나
+ 부정·비교·다중 엔터티 문제 없음
```

현재 25개다. 예시는 다음과 같다.

```text
PCP4, which is a cerebellar Purkinje cell marker

CellMarker candidate: Purkinje cell - PCP4
```

HIGH_CONFIDENCE도 자동 생성 결과이므로 사람이 확인한 후에만 gold로 채택한다.

### NEEDS_REVIEW

자동으로 관계를 확정하기 어려운 문장이다. 현재 848개이며 세부 사유는 다음과 같다.

```text
marker 또는 cell만 부분적으로 일치                 705개
여러 CellMarker pair 후보가 존재                    76개
GPT/Opus 엔터티가 여러 개이거나 서로 불일치         35개
상위 cell_name_class만 일치                          18개
부정 또는 비교 표현이 섞인 문장                     10개
제한적인 cell 약어로 일치                             4개
```

부정 표현은 자동 REJECTED로 처리하지 않는다.

예를 들어 다음 문장에서 `not`만 보고 전체 문장을 탈락시키면 안 된다.

```text
Genes enriched in DC but not in macrophages included Flt3.
```

가능한 해석은 다음과 같다.

```text
Dendritic cell - Flt3: POS
Macrophage - Flt3: NEG
```

따라서 부정·비교 문맥은 사람이 실제 관계 방향을 확인한다.

### REJECTED

현재 807개이며 자동 판정 사유는 다음과 같다.

```text
동일 PMID 안에서 CellMarker pair 표현을 찾지 못함  801개
PMID는 있지만 usable한 cell-marker 관계가 없음       6개
```

REJECTED 역시 최종 탈락 판정이 아니다. 동의어, 유전자 alias, 문장 표기 차이 때문에 자동 검색이 놓친 관계가 있을 수 있으므로 최소한 표본 검수가 필요하다.

## 7. 후보 파일의 컬럼

주요 컬럼은 다음과 같다.

| Column | Meaning |
| --- | --- |
| `row_id`, `PMID`, `sentence` | 원본 후보 문장의 식별자와 내용 |
| `status` | 자동 분류 상태 |
| `reason` | 자동 분류 사유 |
| `candidate_pair_count` | 동일 문장에서 발견된 CellMarker pair 수 |
| `candidate_pairs` | 발견된 cell-marker 후보와 매칭 근거의 JSON 목록 |
| `matched_marker_terms` | 문장에서 발견된 CellMarker marker 목록 |
| `matched_cell_terms` | 문장에서 발견된 CellMarker cell 목록 |
| `has_positive_cue` | 양성 관계 표현 탐지 여부 |
| `has_negative_cue` | 부정 표현 탐지 여부 |
| `has_comparison_cue` | 비교 표현 탐지 여부 |
| `has_entity_ambiguity` | GPT/Opus 엔터티의 다중성 또는 불일치 여부 |
| `review_decision` | 사람이 입력할 `ACCEPT`, `REJECT`, `MODIFY` 결정 |
| `reviewed_pairs` | 사람이 확정한 cell-marker 관계의 JSON 목록 |
| `review_notes` | 판단 근거와 수정 사항 |

## 8. 수동 검수 방법

검수 우선순위는 다음과 같다.

```text
1. HIGH_CONFIDENCE 25개 전수 확인
2. NEEDS_REVIEW 848개 전수 확인
3. REJECTED 807개 중 표본 확인
```

검수자는 문장과 `candidate_pairs`를 함께 보고 다음 값을 입력한다.

```text
review_decision = ACCEPT
  후보 관계가 문장에서 실제 양성/음성 관계로 표현됨

review_decision = REJECT
  같은 PMID이지만 해당 관계가 이 문장에는 없음

review_decision = MODIFY
  관계는 있지만 자동 후보의 cell, marker 또는 polarity 수정이 필요함
```

`reviewed_pairs`에는 최종 승인한 관계만 기록한다.

```json
[
  {"cell_name": "Myoepithelial cell", "symbol": "Cnn1", "relation_label": "POS"},
  {"cell_name": "Myoepithelial cell", "symbol": "Actg2", "relation_label": "POS"}
]
```

한 문장에 여러 관계가 있으면 전부 기록해야 한다. 하나만 기록하면 모델이 추출한 다른 실제 관계가 FP로 잘못 계산될 수 있다.

## 9. 최종 gold test set 생성

수동 검수가 끝나면 `reviewed_pairs`를 펼쳐 한 행에 하나의 관계만 저장한다.

```csv
row_id,PMID,sentence,gold_cell,gold_marker,relation_label
969,28576768,"Myoepithelial cells expressed Cnn1 and Actg2.",Myoepithelial cell,Cnn1,POS
969,28576768,"Myoepithelial cells expressed Cnn1 and Actg2.",Myoepithelial cell,Actg2,POS
```

이 파일부터 최종 gold test set으로 부른다.
현재의 `cellmarker_sentence_gold_candidates.csv`는 gold가 아니라 gold 생성을 위한 검수 후보 파일이다.

## 10. 모델 평가

최종 gold와 `results/full/mapped/prompt3_ontology_mapped.csv`의 예측 관계를 PMID와 Evidence 문장 기준으로 연결한다.

예를 들어 다음과 같다고 가정한다.

```text
Gold
Myoepithelial cell - Cnn1
Myoepithelial cell - Actg2

Model prediction
Myoepithelial cell - Cnn1
Myoepithelial cell - KRT14
```

평가 결과는 다음과 같다.

```text
TP: Myoepithelial cell - Cnn1
FP: Myoepithelial cell - KRT14
FN: Myoepithelial cell - Actg2
```

이 결과로 pair-level precision, recall, F1을 계산한다.
표준화 성능을 별도로 분석하려면 원문 표현 기준 평가와 ontology/official symbol 기준 평가를 구분한다.

## 11. 현재 진행 상태

```text
[완료] all_cell_marker.txt가 single_cell_marker.txt를 포함하는지 확인
[완료] 4,588개 문장 중 PMID가 겹치는 1,680개 선정
[완료] 동일 PMID의 CellMarker 관계를 문장에 역매칭
[완료] HIGH_CONFIDENCE / NEEDS_REVIEW / REJECTED 자동 분류
[현재] 자동 분류 결과 수동 검수 준비
[다음] review_decision과 reviewed_pairs 작성
[다음] 승인 관계를 pair-level 최종 gold로 변환
[마지막] 기존 추출·표준화 결과와 비교 평가
```

## 12. 평가 범위

이 평가가 측정하는 대상은 다음과 같다.

> 4,588개 refined positive 후보 문장 중 CellMarker와 PMID가 겹치는 문장에서, CMAgent가 cell-marker 관계를 얼마나 정확하게 추출하고 표준화했는가.

이 평가는 문장 POS/NEG 분류 성능이나 전체 논문 검색 성능을 직접 평가하지 않는다.
또한 CellMarker에 PMID가 없는 2,908개 문장에 대해서는 정답 여부를 판정하지 않는다.

## 13. 재실행

자동 gold 후보 생성은 다음 명령으로 재실행한다.

```bash
python src/evaluation/build_sentence_gold_candidates.py
```

기본 입력과 출력은 다음과 같다.

```text
Input sentences:
data/processed/refined_positive_sentence_candidates_4588.csv

CellMarker reference:
data/raw/Cell_marker_db/all_cell_marker.txt

Output candidates:
data/processed/cellmarker_sentence_gold_candidates.csv
```
