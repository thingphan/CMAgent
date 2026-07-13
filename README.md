# CMAgent Sentence-level Gold Test Set

## 목적

만들고자 하는 정답지는 다음 질문에 답한다.

> 이 문장을 CMAgent에 입력했을 때 어떤 cell-marker 쌍이 추출되어야 하는가?

CellMarker는 "이 PMID의 논문에 이 cell-marker 관계가 있다"는 문서 단위 정답이다.
CMAgent는 문장에서 관계를 추출하므로 CellMarker 관계가 실제로 등장하는 문장을 찾아 문장 단위 정답으로 바꿔야 한다.

## 입력 데이터

### 평가 문장

`data/processed/refined_positive_sentence_candidates_4588.csv`

- 후보 문장 4,588개
- 고유 PMID 570개

### CellMarker 정답 원본

`data/raw/Cell_marker_db/all_cell_marker.txt`

- 전체 2,537,570행
- 고유 PMID 10,940개
- `single_cell_marker.txt`의 모든 고유 행과 관계를 포함한다.
- 따라서 별도 병합 없이 `all_cell_marker.txt`만 사용한다.

## 처리 과정

```text
4,588개 후보 문장
        |
        | all_cell_marker.txt와 PMID 비교
        v
PMID가 겹치는 문장
        |
        | 동일 PMID의 CellMarker 행만 검색
        v
cell_name + marker/symbol이 모두 문장에 있는 pair 선택
        |
        | 한 행에 하나의 cell-marker pair 저장
        v
수동 ACCEPT / REJECT
        |
        v
최종 sentence-level gold test set
```

## 1. PMID가 겹치는 문장 선택

```text
전체 후보 문장                         4,588개
후보 문장의 고유 PMID                    570개
CellMarker와 겹치는 PMID                 165개
PMID가 겹치는 후보 문장                1,680개
```

1,680개 중 6개는 해당 PMID에 usable한 `cell_name-symbol` 관계가 없다.
따라서 실제 키워드 검색 대상은 1,674개 문장이다.

PMID가 겹치지 않는 문장을 NEG로 판단해서는 안 된다.
CellMarker만으로는 정답 여부를 알 수 없으므로 이번 평가에서 제외한다.

## 2. 동일 PMID 안에서 pair 검색

각 문장에 대해 동일 PMID의 CellMarker 행만 가져온다.

예를 들어 문장이 다음과 같다고 가정한다.

```text
PMID: 12345
Sentence: CD8 T cells strongly expressed PDCD1.
```

동일 PMID의 CellMarker 행이 다음과 같다면 정답 후보로 선택한다.

```text
PMID: 12345
cell_name: CD8+ T cell
symbol: PDCD1
```

문장에 `CD8 T cells`와 `PDCD1`이 모두 있으므로 다음 pair를 생성한다.

```text
CD8+ T cell - PDCD1
```

중요한 조건은 cell과 marker가 **동일한 CellMarker 행의 관계**여야 한다는 것이다.
같은 PMID의 서로 다른 두 행에서 cell과 marker를 따로 가져와 임의로 조합하지 않는다.

## 3. 적용하는 최소 정규화

현재는 의미 추론이나 복잡한 동의어 사전을 사용하지 않는다.

- 대소문자를 무시한다.
- `cell`과 `cells`를 동일하게 처리한다.
- `CD8+`, `CD8 +`, `CD8`의 `+` 공백 차이를 제거한다.
- 하이픈, 밑줄, 슬래시를 공백으로 정규화한다.
- marker는 CellMarker의 `symbol` 또는 `marker` 필드를 사용한다.

현재 사용하지 않는 기능은 다음과 같다.

- Cell Ontology 전체 동의어
- HGNC/NCBI gene alias
- LLM 관계 판정
- 자동 HIGH/REVIEW/REJECTED 분류
- 부정·비교 표현 자동 판정

이 기능들은 첫 결과가 너무 적거나 수동 검수에서 필요성이 확인될 때만 추가한다.

## 4. 현재 키워드 매칭 결과

```text
usable PMID가 있는 문장                  1,674개
cell-marker pair가 하나 이상 잡힌 문장     113개
검수할 sentence-pair 후보                  236개
후보가 잡힌 고유 PMID                       60개
```

문장 수는 1,674개에서 113개로 많이 줄었다.
하지만 236개 pair는 사람이 전수 검수할 수 있는 규모다.

이 감소는 "나머지 문장에 관계가 없다"는 뜻이 아니다.
CellMarker 표준명과 논문 표현이 다르거나 alias가 사용되어 단순 키워드 매칭에서 누락됐을 수 있다.

## 5. 자동 생성되는 후보 파일

`data/processed/cellmarker_sentence_gold_candidates.csv`

한 문장에 여러 관계가 있으면 같은 `row_id`로 여러 행이 생성된다.

```csv
row_id,PMID,sentence,gold_cell,gold_marker,review_decision
969,28576768,"Myoepithelial cells expressed Cnn1 and Actg2.",Myoepithelial cell,Cnn1,
969,28576768,"Myoepithelial cells expressed Cnn1 and Actg2.",Myoepithelial cell,Actg2,
```

전체 컬럼은 다음과 같다.

| Column | Meaning |
| --- | --- |
| `row_id`, `PMID`, `sentence` | 원본 후보 문장 |
| `gold_cell` | CellMarker의 `cell_name` |
| `gold_marker` | CellMarker의 `symbol` |
| `cell_match_text` | 정규화 후 문장에서 매칭된 cell 표현 |
| `marker_match_field` | `symbol` 또는 `marker` 중 어떤 필드로 매칭됐는지 |
| `marker_match_text` | 정규화 후 매칭된 marker 표현 |
| `review_decision` | 사람이 입력할 `ACCEPT` 또는 `REJECT` |
| `review_notes` | 판단 근거 또는 수정 사항 |

## 6. 수동 검수

236개 pair를 한 행씩 읽고 다음 중 하나를 입력한다.

```text
ACCEPT
문장에서 해당 cell-marker 관계가 실제로 표현됨

REJECT
두 키워드는 있지만 서로 관계가 없거나 잘못 연결됨
```

예를 들어 다음 문장은 자동으로 두 pair가 생성될 수 있다.

```text
Myoepithelial cells expressed Cnn1 and Actg2.
```

검수 결과는 다음과 같다.

```text
Myoepithelial cell - Cnn1  ACCEPT
Myoepithelial cell - Actg2 ACCEPT
```

반면 cell과 marker가 우연히 같은 문장에만 존재하면 `REJECT`한다.

## 7. 최종 gold test set

검수가 끝난 뒤 `review_decision == ACCEPT`인 행만 선택한다.
이 행들이 "이 문장에서 추출되어야 하는 정답 관계"가 된다.

```csv
row_id,PMID,sentence,gold_cell,gold_marker
969,28576768,"Myoepithelial cells expressed Cnn1 and Actg2.",Myoepithelial cell,Cnn1
969,28576768,"Myoepithelial cells expressed Cnn1 and Actg2.",Myoepithelial cell,Actg2
```

이 파일을 이미 생성한 모델 추출·표준화 결과와 비교해 TP, FP, FN을 계산한다.

```text
Gold:  Myoepithelial cell - Cnn1
Model: Myoepithelial cell - Cnn1
Result: TP
```

## 현재 진행 상태

```text
[완료] all_cell_marker.txt가 single_cell_marker.txt를 포함하는지 확인
[완료] PMID가 겹치고 usable한 1,674개 문장 선정
[완료] 동일 PMID의 cell-marker pair 키워드 매칭
[완료] 113개 문장에서 236개 pair 후보 생성
[현재] 236개 pair 수동 ACCEPT / REJECT 검수
[다음] ACCEPT 행만 모아 최종 gold test set 생성
[마지막] 기존 추출·표준화 결과 평가
```

## 재실행

```bash
python src/evaluation/build_sentence_gold_candidates.py
```

입출력 파일은 다음과 같다.

```text
Sentences:
data/processed/refined_positive_sentence_candidates_4588.csv

CellMarker:
data/raw/Cell_marker_db/all_cell_marker.txt

Output:
data/processed/cellmarker_sentence_gold_candidates.csv
```
