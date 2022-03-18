# News_Reuters

로이터통신 뉴스를 크롤링하여, <br>
Google Transrator 를 사용하여 한국어로 번역하는 py 파일 입니다. <br>
뉴스탭 Front에 적용하기 위해 news data 크롤링하는 파일입니다.

<img src="https://github.com/hellomungi/News_Reuters/blob/main/news_reuters.JPG"/>


```
import time
import pandas as pd
import requests
from bs4 import BeautifulSoup
import json
import re

from googletrans import Translator
from sqlalchemy import create_engine

def reuters(subject):
    ###################################################################################################################################################
    translator = Translator()

    ## HTTP GET Request
    req = requests.get(str(subject))
    ## HTML 소스 가져오기
    html = req.text
    ## BeautifulSoup으로 html소스를 python객체로 변환하기
    ## 첫 인자는 html소스코드, 두 번째 인자는 어떤 parser를 이용할지 명시.
    ## 이 글에서는 Python 내장 html.parser를 이용했다.
    soup = BeautifulSoup(html, 'html.parser')

    ## html소스중 <script> 부분을 찾음
    script = soup.find('script',{'id':'fusion-metadata'})
    script = str(script)

    ## news를 저장할 Dataframe 생성 , column생성
    news_df = pd.DataFrame()
    news_df['time'] = []
    news_df['url'] = []
    news_df['title'] = []
    news_df['dis'] = []
    news_df['dis_kr'] = []

    ## script 몇번째 문자열부터 읽을지 가늠하는 변수
    a = 0

    ## dataframe index 번호
    df_num = 0
    for i in range(5000):
        ## script[a:-1] 시작시 script[0:-1] 부터 읽으며, for_script가 0보다 커지면 800씩 증가한여,
        ## 다음번 기사는 script[800:-1] 부터 읽기 시작한다.

        for_default_script = script[a:-1]
        for_script = for_default_script.find('canonical_url')
        if for_script > 0:
            a = a + 800
        else:
            break

        ## news 시작지점 = news_url_s, news_title_s 종료지점 = news_url_e, news_title_e
        default_news = for_default_script[for_script:for_script+800]
        news_url_s = default_news.find('canonical_url')
        news_url_e = default_news.find(',"title"')
        news_title_s = default_news.find(',"title"')
        news_title_e = default_news.find(',"description"')
        new_dis_s = default_news.find(',"description"')
        new_dis_e = default_news.find(',"content_code"')
        new_time_s = default_news.find(',"display_time":')
        new_time_e = default_news.find(',"primary_tag":')

        ## 받아온 news에 불필요한 내용은 replace로 치환하여 가공한다
        news_df.loc[df_num, 'time'] = default_news[new_time_s:new_time_e].replace(',"display_time":', '').replace('"', '').replace("\\","")
        news_df.loc[df_num, 'url'] = "https://www.reuters.com/" + default_news[news_url_s:news_url_e].replace('canonical_url":', '').replace('"', '').replace("\\","")
        news_df.loc[df_num, 'title'] = default_news[news_title_s:news_title_e].replace(',"subtitle":null', '').replace(',"title":', '').replace('"', '').replace("\\","")
        dis = default_news[new_dis_s:new_dis_e].replace(',"description":', '').replace('"', '').replace(',headline_feature:GLOBAL MARKETS','').replace("\\","")
        news_df.loc[df_num, 'dis'] = dis

        ## for문 돌때마다 dataframe index 번호를 1씩 늘려줌
        df_num = df_num + 1

    ## news_df 에 중복 뉴스를 duplicates 옵션으로 제거한다.
    news_df = news_df.drop_duplicates()
    news_df = news_df.reset_index(drop=True)
    ndf_dil = news_df.index.tolist()

    for ndi in ndf_dil:
        ## title과 discription 내용을 한국어로 번역하고 저장한다
        title = news_df.loc[ndi, 'title']
        dis = news_df.loc[ndi, 'dis']
        korean = translator.translate(dis,  dest='ko')
        news_df.loc[ndi, 'dis_kr'] = korean.text
        title_kr = translator.translate(title,  dest='ko')
        news_df.loc[ndi, 'title_kr'] = title_kr.text
        print(ndi, " / ", len(ndf_dil), " / ", title_kr.text, korean.text)


    ## news timezone을 Asia로 변경 한다.
    news_df['time'] = pd.to_datetime(news_df.time)
    news_df['time'] = news_df['time'].dt.tz_convert('Asia/Seoul')
    print(news_df)

    ## 뉴스를 DB에 저장한다
    engine = create_engine("mysql+pymysql://root:" + "PASSWORD" + "@localhost:3306/DBNAME?charset=utf8", encoding='utf-8')
    news_df.to_sql(name='TABLE_NAME', con=engine, if_exists='replace', index=False)


    ###################################################################################################################################################

if __name__ == '__main__':
    default_url = 'https://www.reuters.com/'
    reuters(default_url+'world')
    reuters(default_url+'markets') ```
