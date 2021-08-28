---

title: "[Python] DCM(DICOM) 파일 with Python"
excerpt: "CT scan 결과인 DCM 파일에 대해 알아보자"
last_modified_at: 2021-08-23

categories: 

 - Python
 
tags:

 - VISION
 - 의료분야
 - DCM
 - Python

# header:
#   overlay_image: https://images.unsplash.com/photo-1501785888041-af3ef285b470?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1350&q=80
#   overlay_filter: 0.5 # 투명도

toc: true
toc_icon: "bars" # 아이콘 설정
toc_sticky: true #목차를 스크롤과 내릴것인지
toc_label: "목차"

---
비전 쪽을 공부하다가 한 번 직접 해보는게 좋을 것 같아서 캐글에서 진행중이었던
[SIIM-FISABIO-RSNA COVID-19 Detection](https://www.kaggle.com/c/siim-covid19-detection)(현재 종료)에 참여해 봤다. ~~참여만...~~  
흉부 X-ray image로부터 폐렴 증상 여부를 detection하는 게 목적이었는데, 시작부터 맞닥드린 문제는 데이터 포맷이었다.  
일반적 이미지 포맷과는 다른  <span style="color:#ff4c4c">**DCM형식**</span>이었다.  
<br/>
- DICOM은 Digital Imaging and Communication in Medicine 의미함  
- 의료용 디지털 영상의 저장, 출력 및 전송에 사용되는 여러가지 표준을 총칭
- CT나 X-ray, 초음파, MRI등의 의료 image 결과물
- 따로 viewer가 필요함
- - -
## DICOM 파일 읽기 및 jpg/png 저장(python)
pydicom library를 사용하면 dicom파일을 python에서 다룰 수 있음
- pydicom 라이브러리 설치
```ruby
!pip install pydicom
```
- pydicom으로 dcm 데이터 읽기

```ruby
import pydicom as dcm

# 1. 모든 데이터 불러오기 (둘 다 가능)
#raw_file = dcm.dcmread('filename.dcm')
raw_file = dcm.read_file('/content/AI/DCM/51759b5579bc.dcm')
 
# 2. Header 정보 불러오기 (원하는 헤더명 띄어쓰기 없이 적으면 됨)
date = raw_file.StudyDate
 
# 3. Image Array 불러오기
image = raw_file.pixel_array
```
- 불러온 이미지 보기
```ruby
import matplotlib.pyplot as plt
plt.imshow(image, cmap='gray')
```

<p align="center">
<img src="https://user-images.githubusercontent.com/73866596/131220379-453157ac-b35d-44f9-a7d1-37421c3e8b04.png"><br/>
<em>원본을 바로 시각화</em>
</p>

- 다음 코드는 dcm파일을 png/jpg로 저장하기 위해 rescaling하는 함수  
(캐글의 노트북 코드( https://www.kaggle.com/whitegg/dcm-to-png-jpg-using-cv2-simpleitk)에서 가져옴)

```ruby
from PIL import Image
import pandas as pd

import numpy as np
import pydicom
from pydicom.pixel_data_handlers.util import apply_voi_lut

def read_xray(path, voi_lut = True, fix_monochrome = True):
    # Original from: https://www.kaggle.com/raddar/convert-dicom-to-np-array-the-correct-way
    dicom = pydicom.read_file(path)
    
    # VOI LUT (if available by DICOM device) is used to transform raw DICOM data to 
    # "human-friendly" view
    if voi_lut:
        data = apply_voi_lut(dicom.pixel_array, dicom)
    else:
        data = dicom.pixel_array
               
    # depending on this value, X-ray may look inverted - fix that:
    if fix_monochrome and dicom.PhotometricInterpretation == "MONOCHROME1":
        data = np.amax(data) - data
        
    data = data - np.min(data)
    data = data / np.max(data)
    data = (data * 255).astype(np.uint8)
        
    return data


def resize(array, size, keep_ratio=False, resample=Image.LANCZOS):
    # Original from: https://www.kaggle.com/xhlulu/vinbigdata-process-and-resize-to-image
    im = Image.fromarray(array)
    
    if keep_ratio:
        im.thumbnail((size, size), resample) #resize와 다르게 원본 비율이 유지됨
    else:
        im = im.resize((size, size), resample)
    
    return im
```
- **PIL**(Python Image Library)는 이미지 분석 및 처리를 쉽게 할 수 있게 도와주는 라이브러리  
이미지 열기, 자르기, 회전, 상하좌우대칭, 필터링, 합치기, 저장 등 이미지 프로세싱을 쉽게 할수 있음  
- **read_xray**는  dcm 파일을 일고 넘파이 array로 변환하여 반환  
voi_lut은
- **resize**는 이미지를 크기를 변경하는 함수  
\* Image.LANCZOS : 출력 값에 기여할 수 있는 모든 픽셀에서 고품질 필터 Lanczos 필터 (잘린 sinc) 사용(resize() 와 thumbnail()에서만 사용할 수 있음)  
**Lanczos**는 이미지 리사이즈 방법중에 하나로 인코딩 속도는 느리지만 선명한 화질을 보여줌 > 좀 더 상세한 내용은 필요하면 추후 공부  
<br/>

- png/jpg로 저장
```ruby
import os
# png로 저장
dirname = '/content/AI/DCM/51759b5579bc.dcm'
xray = read_xray(dirname)
im = resize(xray, size=512)  
filename = dirname.split('/')[-1][:-4] + '.png' #dirname에서 /기준으로 나누고 -1(뒤에서 첫번째) 선택후 마지막 4글자전까지만 슬라이싱후 .png더하기
#study = dirname.split('/')[-2] + '_study'
im.save(os.path.join(save_dir, filename))
```

<p align="center">
<img src="https://user-images.githubusercontent.com/73866596/131220950-0c31f44e-9220-43d9-ab77-625bbcd12120.png"><br/>
<em>512사이즈 resize하여 png로 저장한 이미지</em>
</p>

- - -
이번에는 간단히 dcm파일을 읽어드리고 데이터로 활용하기 위해 jpg/png파일로 변경하는 정도까지만 알아봤다  
조사하면서 실제로는 의료파일이기 때문에 환자정보를 어떻게 처리할 것인가에 대한 내용이나, 이미지를 어떻게 변환할 것인지에 대한 내용도 있었지만 이 부분은 추후 필요하면 더 알아봐야 겠다
- - -
### 참고  
- https://wezard4u.tistory.com/2895
- https://ballentain.tistory.com/53
- https://jayeon8282.tistory.com/2
- https://github.com/vuno-bmkim/dicom
- https://carmack-kim.tistory.com/100
- https://appia.tistory.com/360
- https://ddolcat.tistory.com/690