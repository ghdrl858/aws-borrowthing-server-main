<h1 align="center"> π λ²λ‘μ°μ½ AWS SERVER</h1>

## π Description

β μ§μ­ μ£Όλ―Όκ° λ¬Όκ±΄μ λΉλ €μ£Όκ³  λ°μ μ μλ κ±°λ μ±μλλ€.

β λμ λλ€λ₯Ό μ€μ νκ³ , νλ λ²μλ₯Ό μ νλ©΄ κ·Έ κ±°λ¦¬ λ΄μ μλ λλ€μ κΈμ λ³Ό μ μμ΅λλ€. (**Naver Maps API - Reverse Geocoding**)

β κ±°λ νμ μ 3κ° μ΄μ λ±λ‘νλ©΄ μΆμ²νλ μνμ λ³Ό μ μμ΅λλ€. (**Item Based Collaborative Filterling** )

β κ±°λ κΈμ μμ±ν  λ μ¬μ§μ λ±λ‘νλ©΄ κ°μ²΄ νμ§λ₯Ό ν΅ν΄ μλμΌλ‘ νκ·Έκ° μμ±λ©λλ€. (**AWS boto3 Rekognition**)

β κ±°λ λΉμ¬μ κ° μ€μκ° μ±νμ ν  μ μμ΅λλ€. (**Firebase Realtime Database**)

<br>

## π Region Dataset Source
β λνλ―Όκ΅­ νμ κ΅¬μ­λ³ μκ²½λ μ’ν

 π μΆμ² : https://skyseven73.tistory.com/23

<br>

##
## π  Environment

β Language
- backend : python 3.8
- android : java, xml

β Tool
- backend : Visual Studio Code, MySQL Workbench, Postman
- Android : Android Studio
- Collaboration : Slack, Postman, Github

β Deployment
- Serverless Flask Framework(AWS Lambda)
- Storage : AWS S3
- Database : AWS RDS

## πΌ Object Detection, Translation
β AWS - boto3 Rekognition (Object Detection) 

β Naver - Papago API(Translatlation)λ₯Ό μ΄μ©ν μλ νκ·Έ κΈ°λ₯ κ΅¬ν

- AWS - boto3 Rekognition (Object Detection) 

```python
# rekognition μ μ΄μ©ν΄μ object detection νλ€.
        client = boto3.client('rekognition',
                            'ap-northeast-2',                               # region
                            aws_access_key_id = Config.ACCESS_KEY,          # ACCESS_KEY   
                            aws_secret_access_key = Config.SECRET_ACCESS)   # SECRET_ACCESS
```
```python
        if 'photo' in request.files:
            # S3μ νμΌ μλ‘λ
            # ν΄λΌμ΄μΈνΈλ‘λΆν° νμΌμ λ°μμ¨λ€.
            files = request.files.getlist("photo")
            for file in files :
                # νμΌλͺμ μ°λ¦¬κ° λ³κ²½ν΄ μ€λ€.
                # νμΌλͺμ, μ λν¬νκ² λ§λ€μ΄μΌ νλ€.
                current_time = datetime.now()
                new_file_name = current_time.isoformat().replace(':', '_') + ('.jpg')

                # μ μ κ° μ¬λ¦° νμΌμ μ΄λ¦μ λ΄κ° λ§λ  νμΌλͺμΌλ‘ λ³κ²½
                file.filename = new_file_name
                s3 = boto3.client('s3', 
                            aws_access_key_id = Config.ACCESS_KEY,
                            aws_secret_access_key = Config.SECRET_ACCESS)

                try :
                    s3.upload_fileobj(file,             # μλ‘λ νμΌ
                                    Config.S3_BUCKET,   # λ²ν· url
                                    file.filename,      # νμΌλͺ
                                    ExtraArgs = {'ACL' : 'public-read', 'ContentType' : file.content_type})    # κΆν, νμ

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
                    # label['Name'] μ΄ κ°μ μ°λ¦¬λ νκ·Έ μ΄λ¦μΌλ‘ μ¬μ©ν κ²
                    try :
                        # ννκ³  λ²μ­νκΈ°
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


## πΌ Recommendation System

β Item Based Collaborative Filtering (μμ΄ν κΈ°λ° νμ νν°λ§)
- μ μ¬λκ° λμ νλ§€μμ μνμ μΆμ²νμ΅λλ€,
``` python
        #  μΆμ²μ μν μκ΄κ³μλ₯Ό μν΄, λ°μ΄ν°λ² μ΄μ€μμ
        #  μ΄ μ μ μ λ³μ  μ λ³΄λ₯Ό, λλΉμμ κ°μ Έμ¨λ€. 
        try :
            connection = get_connection()

            # μμ±μμ κ²μκΈμ΄ μ ν¨νμ§ νμΈνλ€.
            query = '''select * from evaluation_items
                    where authorId = %s;'''
            record = (userId, )
            cursor = connection.cursor(dictionary = True)
            cursor.execute(query, record)
            items = cursor.fetchall()

            if len(items) < 3 :
                cursor.close()
                connection.close()
                return {'error' : 'λ¦¬λ·°λ₯Ό λ¨κΈ΄ νμκ° 3ν λ―Έλ§μλλ€.'}, 400

            # μ μ²΄ νλ§€μμ λ³μ  λ¦¬μ€νΈ
            query = '''select ei.authorId, ei.goodsId, ei.score, g.sellerId 
                from evaluation_items ei
                join goods g
                on ei.goodsId = g.id;'''
                       
            # select λ¬Έμ, dictionary = True λ₯Ό ν΄μ€λ€.
            cursor = connection.cursor(dictionary = True)

            cursor.execute(query)

            # select λ¬Έμ, μλ ν¨μλ₯Ό μ΄μ©ν΄μ, λ°μ΄ν°λ₯Ό κ°μ Έμ¨λ€.
            sellerList = cursor.fetchall()
            
            cursor.close()

            # μ μ  λ³μ  λ¦¬μ€νΈ
            query = '''select ei.authorId, ei.goodsId, ei.score, g.sellerId 
                from evaluation_items ei
                join goods g
                on ei.goodsId = g.id and ei.authorId = %s;'''
            
            record = (userId,)

            # select λ¬Έμ, dictionary = True λ₯Ό ν΄μ€λ€.
            cursor = connection.cursor(dictionary = True)

            cursor.execute(query, record)

            # select λ¬Έμ, μλ ν¨μλ₯Ό μ΄μ©ν΄μ, λ°μ΄ν°λ₯Ό κ°μ Έμ¨λ€.
            items = cursor.fetchall()

            cursor.close()
            connection.close()

        except mysql.connector.Error as e :
            print(e)
            cursor.close()
            connection.close()

            return {"error" : str(e), 'error_no' : 20}, 503

        # νΌλ΄ νμ΄λΈ ν ν
        # μκ΄κ³μ λ§€νΈλ¦­μ€λ‘ λ§λ€κΈ°
        seller_rating_df = pd.DataFrame(sellerList)
        matrix = seller_rating_df.pivot_table(values = 'score', index = 'authorId', columns = 'sellerId', aggfunc = 'mean')
        df = matrix.corr()     
        
        # λλΉλ‘ λΆν° κ°μ Έμ¨, λ΄ λ³μ  μ λ³΄λ₯Ό, λ°μ΄ν°νλ μμΌλ‘
        # λ§λ€μ΄ μ€λ€.
        df_my_rating = pd.DataFrame(data=items)
        

        # μΆμ² νλ§€μλ₯Ό μ μ₯ν , λΉ λ°μ΄ν°νλ μ λ§λ λ€.
        similar_seller_list = pd.DataFrame()

        for i in range(  len(df_my_rating)  ) :
            # λ΄κ° νμ μ λ¨κΈ΄ νλ§€μμ λ€λ₯Έ νλ§€μλ€κ³Όμ μκ΄κ΄κ³λ₯Ό κ΅¬νκ³ , Nanμ μ κ±° νλ€.
            # λ°μ΄ν° νλ μ ννλ‘ λ³κ²½νλ€.
            similar_seller = df[df_my_rating['sellerId'][i]].dropna().sort_values(ascending=False).to_frame()
            # μ»¬λΌλͺμ Correlation λ‘ μ€μ νλ€.
            similar_seller.columns = ['Correlation']
            # Weight μ»¬λΌμ λ§λ€κ³  κ·Έ κ°μ λ΄κ° κ·Έ νλ§€μμκ² μ£Όμλ μ μ * Correlation κ°μΌλ‘ νλ€.
            similar_seller['weight'] = df_my_rating['score'][i] * similar_seller['Correlation']
            # κ·Έλ κ² λ§λ  similar_sellerλ₯Ό concat ν¨μλ₯Ό μ΄μ©ν΄ λΆμ¬μ€λ€.
            similar_seller_list = pd.concat([similar_seller_list, similar_seller])


        # weight μμΌλ‘ μ λ ¬νλ€.
        similar_seller_list.reset_index(inplace=True)

        similar_seller_list = similar_seller_list.groupby('sellerId')['weight'].max().sort_values(ascending=False)

        similar_seller_list = similar_seller_list.reset_index()

        recommened_seller_list = similar_seller_list['sellerId'].to_list()


        # λ³ΈμΈμ΄ νλ§€μλ©΄ μ κ±°
        if userId in recommened_seller_list :
            recommened_seller_list.remove(userId)

        # νλ§€μ λ¦¬μ€νΈκ° 3λͺ λ³΄λ€ λ§μΌλ©΄ 3λͺ κΉμ§λ§ μ¬μ©
        if len(recommened_seller_list) > 3 :
            recommened_seller_list = recommened_seller_list[:2+1]
```
<br>

### URL
- νμ΄λΈ μ€κ³μ URL : https://www.erdcloud.com/d/nkBL3qYezYH993rSj
- API λͺμΈμ URL : https://documenter.getpostman.com/view/21511146/VUxLwU8Q
- μλλ‘μ΄λ κΉνλΈ URL : https://github.com/ghdrl858/android-borrowthing-final
- νλ‘μ νΈ κΈ°μ μ URL : https://docs.google.com/presentation/d/1j92qjWEymkopXsib6F34sBA_Js4zooMSsCpZFUY8Qy0/edit#slide=id.g35f391192_00
