<h1 align="center"> 🙌 버로우씽 AWS SERVER</h1>

## 📃 Description

✅ 지역 주민간 물건을 빌려주고 받을 수 있는 거래 앱입니다.

✅ 나의 동네를 설정하고, 활동 범위를 정하면 그 거리 내에 있는 동네의 글을 볼 수 있습니다. (**Naver Maps API - Reverse Geocoding**)

✅ 거래 평점을 3개 이상 등록하면 추천하는 상품을 볼 수 있습니다. (**Item Based Collaborative Filterling** )

✅ 거래 글을 작성할 때 사진을 등록하면 객체 탐지를 통해 자동으로 태그가 생성됩니다. (**AWS boto3 Rekognition**)

✅ 거래 당사자 간 실시간 채팅을 할 수 있습니다. (**Firebase Realtime Database**)

<br>

## 📘 Region Dataset Source
✅ 대한민국 행정구역별 위경도 좌표

 👉 출처 : https://skyseven73.tistory.com/23

<br>

##
## 🛠 Environment

✅ Language
- backend : python 3.8
- android : java, xml

✅ Tool
- backend : Visual Studio Code, MySQL Workbench, Postman
- Android : Android Studio
- Collaboration : Slack, Postman, Github

✅ Deployment
- Serverless Flask Framework(AWS Lambda)
- Storage : AWS S3
- Database : AWS RDS

## 💼 Object Detection, Translation
✅ AWS - boto3 Rekognition (Object Detection) 

✅ Naver - Papago API(Translatlation)를 이용한 자동 태그 기능 구현

- AWS - boto3 Rekognition (Object Detection) 

```python
# rekognition 을 이용해서 object detection 한다.
        client = boto3.client('rekognition',
                            'ap-northeast-2',                               # region
                            aws_access_key_id = Config.ACCESS_KEY,          # ACCESS_KEY   
                            aws_secret_access_key = Config.SECRET_ACCESS)   # SECRET_ACCESS
```
```python
        if 'photo' in request.files:
            # S3에 파일 업로드
            # 클라이언트로부터 파일을 받아온다.
            files = request.files.getlist("photo")
            for file in files :
                # 파일명을 우리가 변경해 준다.
                # 파일명은, 유니크하게 만들어야 한다.
                current_time = datetime.now()
                new_file_name = current_time.isoformat().replace(':', '_') + ('.jpg')

                # 유저가 올린 파일의 이름을 내가 만든 파일명으로 변경
                file.filename = new_file_name
                s3 = boto3.client('s3', 
                            aws_access_key_id = Config.ACCESS_KEY,
                            aws_secret_access_key = Config.SECRET_ACCESS)

                try :
                    s3.upload_fileobj(file,             # 업로드 파일
                                    Config.S3_BUCKET,   # 버킷 url
                                    file.filename,      # 파일명
                                    ExtraArgs = {'ACL' : 'public-read', 'ContentType' : file.content_type})    # 권한, 타입

                except Exception as e:
                    return {'error' : str(e)}, 500
```
```python
                response = client.detect_labels(Image = {
                                                'S3Object' : {
                                                        'Bucket' : Config.S3_BUCKET,
                                                        'Name' : file.filename
                                                        }},
                                        MaxLabels = 2)
```

- Naver - Papago API(Translatlation)
```python
                for label in response['Labels'] :
                    # label['Name'] 이 값을 우리는 태그 이름으로 사용할것
                    try :
                        # 파파고 번역하기
                        hearders = {'Content-Type' : 'application/x-www-form-urlencoded; charset=UTF-8',
                            'X-Naver-Client-Id' : Config.NAVER_CLIENT_ID,
                            'X-Naver-Client-Secret' : Config.NAVER_CLIENT_SECRET}

                        data = {'source' : 'en',
                                'target' : 'ko',
                                'text' : label['Name']}

                        res = requests.post(Config.NAVER_PAPAGO_URL, data, headers = hearders)
                        
                        translatedText = res.json()['message']['result']['translatedText']
```

<br>


## 💼 Recommendation System

✅ Item Based Collaborative Filtering (아이템 기반 협업 필터링)
- 유사도가 높은 판매자의 상품을 추천했습니다,
``` python
        #  추천을 위한 상관계수를 위해, 데이터베이스에서
        #  이 유저의 별점 정보를, 디비에서 가져온다. 
        try :
            connection = get_connection()

            # 작성자와 게시글이 유효한지 확인한다.
            query = '''select * from evaluation_items
                    where authorId = %s;'''
            record = (userId, )
            cursor = connection.cursor(dictionary = True)
            cursor.execute(query, record)
            items = cursor.fetchall()

            if len(items) < 3 :
                cursor.close()
                connection.close()
                return {'error' : '리뷰를 남긴 횟수가 3회 미만입니다.'}, 400

            # 전체 판매자의 별점 리스트
            query = '''select ei.authorId, ei.goodsId, ei.score, g.sellerId 
                from evaluation_items ei
                join goods g
                on ei.goodsId = g.id;'''
                       
            # select 문은, dictionary = True 를 해준다.
            cursor = connection.cursor(dictionary = True)

            cursor.execute(query)

            # select 문은, 아래 함수를 이용해서, 데이터를 가져온다.
            sellerList = cursor.fetchall()
            
            cursor.close()

            # 유저 별점 리스트
            query = '''select ei.authorId, ei.goodsId, ei.score, g.sellerId 
                from evaluation_items ei
                join goods g
                on ei.goodsId = g.id and ei.authorId = %s;'''
            
            record = (userId,)

            # select 문은, dictionary = True 를 해준다.
            cursor = connection.cursor(dictionary = True)

            cursor.execute(query, record)

            # select 문은, 아래 함수를 이용해서, 데이터를 가져온다.
            items = cursor.fetchall()

            cursor.close()
            connection.close()

        except mysql.connector.Error as e :
            print(e)
            cursor.close()
            connection.close()

            return {"error" : str(e), 'error_no' : 20}, 503

        # 피봇 테이블 한 후
        # 상관계수 매트릭스로 만들기
        seller_rating_df = pd.DataFrame(sellerList)
        matrix = seller_rating_df.pivot_table(values = 'score', index = 'authorId', columns = 'sellerId', aggfunc = 'mean')
        df = matrix.corr()     
        
        # 디비로 부터 가져온, 내 별점 정보를, 데이터프레임으로
        # 만들어 준다.
        df_my_rating = pd.DataFrame(data=items)
        

        # 추천 판매자를 저장할, 빈 데이터프레임 만든다.
        similar_seller_list = pd.DataFrame()

        for i in range(  len(df_my_rating)  ) :
            # 내가 평정을 남긴 판매자와 다른 판매자들과의 상관관계를 구하고, Nan을 제거 한다.
            # 데이터 프레임 형태로 변경한다.
            similar_seller = df[df_my_rating['sellerId'][i]].dropna().sort_values(ascending=False).to_frame()
            # 컬럼명을 Correlation 로 설정한다.
            similar_seller.columns = ['Correlation']
            # Weight 컬럼을 만들고 그 값을 내가 그 판매자에게 주었던 점수 * Correlation 값으로 한다.
            similar_seller['weight'] = df_my_rating['score'][i] * similar_seller['Correlation']
            # 그렇게 만든 similar_seller를 concat 함수를 이용해 붙여준다.
            similar_seller_list = pd.concat([similar_seller_list, similar_seller])


        # weight 순으로 정렬한다.
        similar_seller_list.reset_index(inplace=True)

        similar_seller_list = similar_seller_list.groupby('sellerId')['weight'].max().sort_values(ascending=False)

        similar_seller_list = similar_seller_list.reset_index()

        recommened_seller_list = similar_seller_list['sellerId'].to_list()


        # 본인이 판매자면 제거
        if userId in recommened_seller_list :
            recommened_seller_list.remove(userId)

        # 판매자 리스트가 3명 보다 많으면 3명 까지만 사용
        if len(recommened_seller_list) > 3 :
            recommened_seller_list = recommened_seller_list[:2+1]
```
<br>

### URL
- 테이블 설계서 URL : https://www.erdcloud.com/d/nkBL3qYezYH993rSj
- API 명세서 URL : https://documenter.getpostman.com/view/21511146/VUxLwU8Q
- 안드로이드 깃허브 URL : https://github.com/fullspringwater/android-borrowthing
- 프로젝트 기술서 URL : https://docs.google.com/presentation/d/174_j53JRkbvM00zhmGBc4ILLhdFF3gLaqkYo_bnpj68/edit#slide=id.g155187810e0_2_5

