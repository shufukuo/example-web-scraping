# example-web-scraping

爬 591 網站上 台北市/新北市 所有出租物件的網頁內容

```python
#
#

from bs4 import BeautifulSoup
from selenium import webdriver
from selenium.webdriver.common.action_chains import ActionChains
import requests as req
import pandas as pd
import time
from datetime import datetime

#

t0 = time.time()

# 宣告 empty DataFrame 儲存租屋物件資料
df = pd.DataFrame( columns = [
        'url',            # 網址
        'contact',        # 出租者/聯絡人
        'is_landlord',    # 是否為屋主
        'phone_number',   # 連絡電話
        'house_type',     # 房屋型態
        'house_kind',     # 房屋現況
        'house_size',     # 房屋坪數
        'house_floor',    # 房屋樓層
        'houseIntro',     # 屋況說明
        'datetime'        # 修改日期時間
        ] )

# 宣告房屋屬性對應欄位 dictionary
dic_house = {
        "坪數": "house_size",
        "樓層": "house_floor",
        "型態": "house_type",
        "現況": "house_kind"
        }

# 搜尋 台北市 ele_city[0] 和 新北市 ele_city[1]
for i in range(0,2):
    t01 = time.time()
    
    t00 = time.time()
    
    # 開啟瀏覽器
    browser = webdriver.Chrome()
    #browser.minimize_window()
    
    # 連線到租屋搜尋頁面
    browser.get("https://rent.591.com.tw/?kind=0&region=1")
    time.sleep(1)
    
    # 點擊取消 popup 視窗
    ele = browser.find_element_by_class_name("search-location-span ")
    action = ActionChains(browser).move_to_element(ele)
    action.click(ele)
    action.perform()
    time.sleep(1)
    
    # 點擊 位置 的 縣市 選單
    ele = browser.find_element_by_class_name("search-location-span ")
    action = ActionChains(browser).move_to_element(ele)
    action.click(ele)
    action.perform()
    
    # 取得 位置 的 縣市 elements
    ele_city = browser.find_elements_by_class_name("city-li")
    
    # ele_city[0]: 台北市,  ele_city[1]: 新北市
    # 點擊縣市
    city_text = ele_city[i].text
    print(city_text)
    action = ActionChains(browser).move_to_element(ele_city[i])
    action.click(ele_city[i])
    action.perform()
    time.sleep(2)
    
    # 宣告 empty list 儲存 //rent.591.com.tw/rent-detail-***.html 連結
    lst_rent_url = []
    
    # 取得網頁原始碼
    page_source = browser.page_source
    
    # 宣告 html.parser 物件
    soup = BeautifulSoup(page_source, "html.parser")
    
    # 搜尋 a 開頭
    find_a = soup.find_all("a")
    
    # 取得第 1 頁 rent-detail urls
    for e in find_a:
        try:
            if "//rent.591.com.tw/rent-detail-" in e["href"]:
                if e["href"] not in lst_rent_url:
                    lst_rent_url.append(e["href"])
        except:
            continue
    
    print("page 1")
    
    # 取得最大頁數
    ele_pageNum = browser.find_elements_by_class_name("pageNum-form")
    ele_pageNum.reverse()
    max_page = int(ele_pageNum[0].text)
    
    # 點擊下一頁並取得每一頁 rent-detail urls
    for p in range (1, max_page):
        # 取得 下一頁 element
        ele_pageNext = browser.find_element_by_class_name("pageNext")
        
        # 點擊下一頁
        action = ActionChains(browser).move_to_element(ele_pageNext)
        action.click(ele_pageNext)
        action.perform()
        time.sleep(0)
        #time.sleep(2)
        
        # 取得網頁原始碼
        # try until page loading completed
        while True:
            try:
                if int(browser.find_element_by_class_name("pageCurrent").text) != p:
                    page_source = browser.page_source
                    break
            except:
                print("error - page loading")
                continue
        
        # 宣告 html.parser 物件
        soup = BeautifulSoup(page_source, "html.parser")
        
        # 搜尋 a 開頭
        find_a = soup.find_all("a")
        
        # 取得第 p 頁 rent-detail urls
        for e in find_a:
            try:
                if "//rent.591.com.tw/rent-detail-" in e["href"]:
                    if e["href"] not in lst_rent_url:
                        lst_rent_url.append(e["href"])
            except:
                continue
        
        print("page " + str(p+1))
    
    browser.quit()
    print( time.time() - t00 )
    print("done - 取得每一頁 rent-detail urls")
    
    t00 = time.time()
    
    # 宣告 DataFrame 儲存租屋物件資料
    n_lst_rent_url = len(lst_rent_url)
    tmp_df = pd.DataFrame( index = range(0, n_lst_rent_url), columns = df.columns )
    
    # 開啟一個新瀏覽器
    browser_rent = webdriver.Chrome()
    
    # 取得各 rent-detail url 內容
    for i_url in range(0, n_lst_rent_url):
        # 連線到租屋頁面
        tmp_url = "https:" + lst_rent_url[i_url].strip()
        browser_rent.get(tmp_url)
        
        # 儲存網址
        tmp_df.loc[i_url, 'url'] = tmp_url
        
        # 檢查網頁是否存在
        try:
            browser_rent.find_element_by_class_name("error-info")
            continue
        except:
            try:
                tmp_ele_text = browser_rent.find_element_by_class_name("avatarRight").text
            except:
                continue
        
        # 取得 出租者姓名
        #tmp_ele_text = browser_rent.find_element_by_class_name("avatarRight").text
        tmp_ele_text_split = tmp_ele_text.split("\n")
        tmp_df.loc[i_url, 'contact'] = tmp_ele_text_split[0]
        
        # 取得 是否為屋主
        if len(tmp_ele_text_split) == 2:
            tmp_df.loc[i_url, 'is_landlord'] = int( "屋主" in tmp_ele_text_split[1] )
        
        # 取得 連絡電話
        #tmp_ele_text = browser_rent.find_element_by_class_name("hidtel").text
        
        # 取得 房屋型態 房屋現況 房屋坪數 房屋樓層
        tmp_ele_text = browser_rent.find_element_by_class_name("attr").text
        tmp_ele_text_split = tmp_ele_text.split("\n")
        for s in tmp_ele_text_split:
            tmp_split = s.split(":")
            if ( dic_house.__contains__(tmp_split[0].strip()) ):
                tmp_df.loc[i_url, dic_house[tmp_split[0].strip()]] = tmp_split[1].strip()
        
        # 取得 屋況說明
        tmp_df.loc[i_url, 'houseIntro'] = browser_rent.find_element_by_class_name("houseIntro").text.replace("\n", "").replace(",", " ")
        
        # 資料取得時間
        tmp_df.loc[i_url, 'datetime'] = datetime.now().strftime("%Y/%m/%d %H:%M:%S")
    
    # append tmp_df to df
    df = df.append(tmp_df, sort=False)
    
    browser_rent.quit()
    print( time.time() - t00 )
    print("done - 取得各 rent-detail url 內容")
    
    print( time.time() - t01 )
    print( "done - " + city_text )


df = df.reset_index()

t00 = time.time()
df.to_csv( "dt.txt", na_rep = "null", index = False )
print( time.time() - t00 )

print( time.time() - t0 )


###
###
###

```
