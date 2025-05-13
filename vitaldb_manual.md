# VitalDB 사용 매뉴얼

## 목차
1. [소개](#소개)
2. [설치 방법](#설치-방법)
3. [Vital File API](#vital-file-api)
4. [Dataset API](#dataset-api)
5. [데이터 변환 및 분석](#데이터-변환-및-분석)
6. [실제 사용 사례](#실제-사용-사례)
7. [보안 고려사항](#보안-고려사항)

## 소개

VitalDB는 수술 환자의 생체신호를 기록한 고해상도 다중 파라미터 데이터베이스입니다. 서울대학교병원에서 2016년 8월부터 2017년 6월까지 6,388명의 비심장 수술 환자를 대상으로 수집된 데이터를 포함하고 있습니다. 이 데이터베이스는 수술 중 환자 모니터링과 관련된 기계학습 연구를 촉진하기 위해 개발되었습니다.

### 주요 특징
- **486,451개**의 파형 및 수치 데이터 트랙(환자당 평균 87개)
- 심전도, 혈압, 산소포화도, 체온 등 **196개**의 수술 중 모니터링 파라미터
- 고해상도 시간 데이터: 수치 데이터 1-7초, 파형 데이터 62.5-500Hz
- 익명화된 데이터로 API 및 Python 라이브러리를 통해 무료로 접근 가능

## 설치 방법

### 필수 요구사항
- Python 3.6 이상

### 설치 명령어
```python
pip install vitaldb
```

### 기본 임포트
```python
import vitaldb
```

## Vital File API

### 파일 다운로드 및 로드

#### 케이스 파일 다운로드
```python
# 케이스 ID가 1인 vital 파일 다운로드
vf = vitaldb.VitalFile(1)
vf.to_vital('1.vital')

# 특정 트랙만 다운로드 (더 빠름)
vf = vitaldb.VitalFile(1, ['SNUADC/ECG_II'])
vf.to_vital('1_ecg.vital')
```

#### 파일 형식별 로드 함수

| 함수 | 설명 | 주요 매개변수 |
|------|------|--------------|
| `read_vital(ipath)` | vital 파일 읽기 | `ipath`: 파일 경로<br>`track_names`: 트랙 이름 목록<br>`header_only`: 헤더만 읽을지 여부 |
| `read_csv(ipath)` | CSV 파일 읽기 | `ipath`: 파일 경로<br>`track_names`: 트랙 이름 목록<br>`interval`: 샘플링 간격 |
| `read_wfdb(ipath)` | WFDB 형식 파일 읽기 | `ipath`: 파일 경로<br>`track_names`: 트랙 이름 목록 |
| `read_parquet(ipath)` | parquet 파일 읽기 | `ipath`: 파일 경로<br>`track_names`: 트랙 이름 목록 |

### 데이터 변환 및 저장

#### 데이터 변환 함수

```python
# NumPy 배열로 변환
samples = vf.to_numpy(['SNUADC/ART'], 1/100)

# Pandas DataFrame으로 변환
df = vf.to_pandas(['SNUADC/ART'], 1/100, return_datetime=True)
```

#### 파일 저장 함수

| 함수 | 설명 | 형식 |
|------|------|------|
| `to_vital(opath)` | vital 파일로 저장 | 바이너리 |
| `to_csv(opath, track_names, interval)` | CSV 파일로 저장 | 텍스트 |
| `to_wfdb(opath, track_names)` | WFDB 형식으로 저장 | 바이너리 |
| `to_wav(opath, track_names, srate)` | 오디오 파일로 저장 | 바이너리 |
| `to_parquet(opath)` | parquet 파일로 저장 | 바이너리 |

## Dataset API

### 데이터셋 검색 및 로드

#### 케이스 찾기
```python
# ECG_II와 ART 트랙이 모두 있는 케이스 ID 찾기
caseids = vitaldb.find_cases(['ECG_II', 'ART'])
print(f"찾은 케이스 수: {len(caseids)}")
```

#### 케이스 데이터 로드
```python
# 첫 번째 케이스의 ECG_II 및 ART 데이터 로드 (100Hz 샘플링)
vals = vitaldb.load_case(caseids[0], ['ECG_II', 'ART'], 1/100)
ecg = vals[:, 0]  # 첫 번째 열 (ECG_II)
art = vals[:, 1]  # 두 번째 열 (ART)
```

## 데이터 변환 및 분석

### 데이터 시각화

```python
import matplotlib.pyplot as plt

# 동맥압 파형 시각화
vf = vitaldb.VitalFile(1, ['SNUADC/ART'])
samples = vf.to_numpy(['SNUADC/ART'], 1/100)

plt.figure(figsize=(20, 5))
plt.plot(samples[:, 0])
plt.title('동맥압 파형')
plt.xlabel('시간 (1/100 초)')
plt.ylabel('압력 (mmHg)')
plt.show()
```

### 데이터 편집 함수

| 함수 | 설명 | 주요 매개변수 |
|------|------|--------------|
| `crop(dtfrom, dtend)` | 특정 시간 범위로 자르기 | `dtfrom`: 시작 시간<br>`dtend`: 종료 시간 |
| `get_track_names()` | 트랙 이름 목록 가져오기 | - |
| `remove_track(dtname)` | 특정 트랙 삭제 | `dtname`: 삭제할 트랙 이름 |
| `add_track(dtname, recs)` | 새 트랙 추가 | `dtname`: 트랙 이름<br>`recs`: 데이터 레코드 |

## 실제 사용 사례

### 사용 사례 1: 심박출량 예측 모델 개발
```python
# 필요한 생체신호 데이터 로드
caseids = vitaldb.find_cases(['ECG_II', 'ART', 'PPG', 'CO'])
training_data = []

for caseid in caseids[:100]:  # 처음 100개 케이스만 사용
    # 데이터 로드 및 전처리
    vals = vitaldb.load_case(caseid, ['ECG_II', 'ART', 'PPG', 'CO'], 1)
    # 모델 훈련에 필요한 전처리 및 특성 추출
    # ...

# 기계학습 모델 훈련
# ...
```

### 사용 사례 2: 수술 중 저혈압 예측
```python
# 마취 시작 후 30분 동안의 데이터를 사용하여 향후 저혈압 발생 예측
caseids = vitaldb.find_cases(['ART', 'Solar8000/SBP', 'Solar8000/HR'])

# 데이터 전처리 및 모델 개발
# ...
```

## 보안 고려사항

1. **데이터 익명화**: VitalDB의 모든 데이터는 익명화되어 있지만, 연구 결과 발표 시 환자 식별 가능성에 주의해야 합니다.

2. **API 키 관리**: API 사용 시 발급받은 키는 안전하게 관리하고 코드에 하드코딩하지 마세요.

3. **데이터 저장**: 다운로드한 데이터는 안전한 저장소에 보관하고 필요시 암호화를 고려하세요.

4. **연구 윤리**: 모든 연구는 관련 기관의 IRB(연구윤리심의위원회) 승인을 받은 후 진행해야 합니다.

---

이 문서는 VitalDB의 기본적인 사용 방법과 주요 기능을 설명합니다. 자세한 내용은 [공식 문서](https://vitaldb.net/docs)를 참조하시기 바랍니다.
