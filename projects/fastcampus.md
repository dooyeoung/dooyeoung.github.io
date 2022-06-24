---
layout: default
is_contact: true
---

# 2018 

1. **2018.08 - 2018.09 > e-commerce 상품 이미지 제작 및 자동등록 서비스** <br>
	 [Python, Jquery]<br>

	상품 기본 정보 입력후 상품 정보 이미지 자동 제작 및 자동등록 기능 [**link**](https://github.com/dooyeoung/e-commerce_util)
	- 상품정보, 영양정보 입력 후 Mysql 저장가능한 관리자 페이지 구축
	- 등록된 이미지 템플릿을 사용하여 상품이미지 제작 기능 개발
	- selenium을 사용한 gmarket 자동등록 기능 개발

1. **2018.07 - 2018.07 > Walmart Store Sales Forecasting**<br>
	 [*Python, Pandas, Matplotlib*]  <br>

	날씨, 매장정보, 유가, 실업률과 주별 판매량 예측  [**link**](https://github.com/dooyeoung/bike_sharing_demand)
	- Data Preprocessing
		- 날씨별, 시간별 데이터 파악 
	- Data Analysis 
		- Gradient boost regressor를 사용하여 분석
		- fbpophet을 사용하여 시계열 분석

1. **2018.05 - 2018.05 > Bike Sharing Demand**<br>
	 [Python, Pandas, Matplotlib]  <br>

	날씨와 시간 데이터를 통해 자전거 대여량 예측  [**link**](https://github.com/dooyeoung/bike_sharing_demand)
	- Data Preprocessing
		- 날씨별, 시간별 데이터 파악 
	- Data Analysis
		- Randomforest를 사용하여 분석
		- Gradient boost regressor를 사용하여 분석
		- Xgboostregressor를 사용하여 분석 

1. **2018.04 - 2018.04 > Recommend coordination** <br>
 	[Python, MySQL, Bootstrap, JQuery] <br>

	웹 코디 이미지 색상 분석을 이용한 coordination 추천  [**link**](https://github.com/dooyeoung/codi_recommendation)
	- 데이터 크롤링 및 데이터베이스 구축
		- Phantomjs로 Mapssi 코디 데이터 크롤링
		- 이미지 주소, 색상 데이터 MySQL 저장
	- 이미지 분석
		- RGB clustering으로 대표색상 추출
		- KNN으로 색상검색기능 구현
		- MLP로 색상추천기능 구현
	- Flask server 구축
		- bootstrap으로 레이아웃 구성
		- RGB값을 KNN을 사용하여 검색기능 구성

1. **2018.03 - 2018.04 > Walmart Trip Type Classification**<br>
 	[Python, Pandas, Matplotlib] <br>

	구매 품목과 수량으로 고객을 38개로 분류  [**link**](https://github.com/dooyeoung/walmart_recruiting_trip_type_classification)
	- Data Preprocessing
		- 상품 Missing value 처리
		- Type, 고객별 구매데이터 취합
	- Data Analysis
		- Xgboost를 사용하여 classification 분석
		- LightGBM을 사용하여 classification 분석

1. **2018.02 – 2018.03 > Walmart : Sales in Stormy Weather** <br>
 	[Python, Pandas, Matplotlib]<br>

	날씨를 토대로 각 매장의 아이템의 판매량을 예측  [**link**](https://github.com/dooyeoung/walmart_recruiting2_stormy_weather)
	- Data Preprocessing
		- weather data missing value 처리
		- train data store 별로 데이터 취합
	- Data Analysis
		- OLS을 이용하여 Regression 분석