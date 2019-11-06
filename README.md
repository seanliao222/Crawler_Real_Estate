# real_estat_crawler
 ＝＝＝實價登錄資料下載與整理＝＝＝
 1. 下載實價登錄資料
 2. 整合所有下載的檔案
 3. 篩選與計算



# In[1]:載入專案所需packages
~~~~{python}
import pandas as pd
import requests
import os
import csv
import time
from os import listdir
import re
~~~~

# In[2]:Definition of Function
~~~~{python}
def real_estate_crawler(year, season,files_name,txn_type):
    #real estate data info
    url = "https://plvr.land.moi.gov.tw//DownloadSeason?season="+str(year)+"S"+str(season)+"&fileName="+str(files_name)+"_lvr_land_"+str(txn_type)+".csv"
    file_dir  = './real_estate/''{}.csv'.format(str(year)+"_"+str(season)+"_"+str(files_name)+"_lvr_land_"+str(txn_type))
    #check download or not
    while not os.path.exists(file_dir):
        res = requests.get(url)
        # 下載所需資料
        if res.status_code == 200 and '系統發生錯誤' not in res.text:
            dir_path = 'real_estate'
            if not os.path.isdir(dir_path):
                os.makedirs(dir_path)
            with open(file_dir, 'w', newline='',encoding='utf-8-sig') as csvfile:
                #部分資料欄位出現錯誤符號導致欄位錯位，直接透過request處理，在寫入CSV
                csvfile.writelines(str(res.text.replace('transaction year, month and day', 'transaction year month and day').replace('/',' ')))
        else: print('查無' + str(year) + '年第'+ str(season) +'季資料')
        break
        
#將路徑下所有檔名寫入陣列                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        
def find_csv_filenames( path_to_dir, suffix=".csv" ):
    filenames = listdir(path_to_dir)
    return [ filename for filename in filenames if filename.endswith( suffix ) ]

#改檔名
def replaceMultiple(OldString, Replaces, NewString):
    for elem in Replaces :
        if elem in OldString :
            OldString = OldString.replace(elem, NewString)    
    return  OldString


#將所有cvs檔案寫入同一個dataframe,並增加新欄位
def add_df_column(SN ,mypath ,filenames):
    replace_list = ['_lvr_land', '.csv']
    file_Catg = replaceMultiple(filenames, replace_list , "")
    file_dir = './'+ mypath +'/' + filenames
    dfs = pd.read_csv(file_dir, sep = ',',header = 0,skiprows = 1)
    #將檔名寫入欄位df_name
    dfs['df_name'] =  file_Catg
    
    #補缺失值避免因為格式判斷錯誤
    dfs.fillna(0, inplace = True)
    #部分資料建築年月不是數字，統一調整為數字資料
    temp_type_ch =[]
    if (dfs['construction to complete the years'].dtypes) == object:
        for key in range(len(dfs['construction to complete the years'])):
            if type(dfs['construction to complete the years'][key]) != float and type(dfs['construction to complete the years'].iloc[key]) != int: 
                temp_type_ch.append(pd.to_numeric(dfs['construction to complete the years'].iloc[key].replace('-','').replace('/','')))
            else:
                temp_type_ch.append(dfs['construction to complete the years'].iloc[key])
        dfs['construction to complete the years'] = temp_type_ch
    
    #因樓層欄位資料部分為文字，統一改為數字以利後續運用        
    temp_data =[]
    for i in range(len(dfs['total floor number'])):
        if type(dfs['total floor number'].iloc[i]) != str:
            temp_data.append(dfs['total floor number'].iloc[i])
        else:
            temp_data.append(trans(dfs['total floor number'].iloc[i].replace('層','')))
    dfs['total floor number'] = temp_data
    return  dfs

#中文數字轉阿拉伯數字
def trans(s):
    num = 0
    digit = {'一': 1, '二': 2, '三': 3, '四': 4, '五': 5, '六': 6, '七': 7, '八': 8, '九': 9}
    if s:
        idx_q, idx_b, idx_s = s.find('千'), s.find('百'), s.find('十')
        if idx_q != -1:
            num += digit[s[idx_q - 1:idx_q]] * 1000
        if idx_b != -1:
            num += digit[s[idx_b - 1:idx_b]] * 100
        if idx_s != -1:
            num += digit.get(s[idx_s - 1:idx_s], 1) * 10
        if s[-1] in digit:
            num += digit[s[-1]]
    return num
~~~~

# In[3]:下載檔案
~~~~{python}
if __name__ == '__main__':
    #台北 新北 高雄
    city_1 = ["A","F","E"]
    #桃園 台中
    city_2 = ["H","B"]
    #開始下載檔案
    for year in range(103, 109):
        for season in range(1,5):
            print('正在下載',year,'年第', season,'季的實價登錄資料')
            for FN1 in range(0,len(city_1),1):
                files_name = city_1[FN1]
                real_estate_crawler(year, season,files_name,"A")
            for FN2 in range(0,len(city_2),1):
                files_name = city_2[FN2]
                real_estate_crawler(year, season,files_name,"B")    
~~~~


# In[4]:整合資料
~~~~{python}
    # 指定要列出所有檔案的目錄
    mypath = 'real_estate'
    filenames = find_csv_filenames(mypath)
    filenames.sort()

    #file append to dataframe
    dataframes = pd.DataFrame()
    for i in range(len(filenames)):
        dfs = add_df_column(i,mypath,filenames[i])
        dfs['the berth total price Yuan'] = dfs['the berth total price Yuan'].astype(float)
        dfs['total price Yuan'] = dfs['total price Yuan'].astype(float)
        dataframes = dataframes.append(dfs,ignore_index = True, sort = False)


    #若已經之前已整合過資料可以再把資料讀出來
    dataframes = pd.read_csv('df_all.csv')
~~~~

# In[5]:篩選資料
~~~~{python}
    # 篩選1.【總樓層數】需【大於等於十三層】2.【建物型態】為【住宅大樓】3.【主要用途】為【住家用】
    df_filter = dataframes[(dataframes['total floor number'] >= 13) &
                          (dataframes['building state'].str.contains('住宅大樓')) &
                          (dataframes['main use'].str.contains('住家用'))]
    #將篩選後的資料寫入CSV，為確保資料用excel打開中文編碼可以正確，編碼改用'utf-8-sig'帶入ＢOM       
    dataframes.to_csv('df_filter.csv', index = False, header = True, encoding='utf-8-sig')
~~~~

# In[7]:計算價格
~~~~{python}
  #計算【總件數】【總車位數】(透過交易筆棟數)【平均總價元】【平均車位總價元】
  #車位要從欄位“交易筆棟數“去擷取
  car_list = []
  for car in range(len(df_filter['transaction pen number'])):
      temp_str = df_filter['transaction pen number'].iloc[car]
      car_list.append(int(temp_str[temp_str.find('車位') + 2:len(temp_str)]))
  #總件數        
  tot_case_cnt = df_filter['serial number'].count()
  #總件數金額
  tot_case_price = sum(df_filter['total price Yuan'])
  #計算車位數 
  car_tot_cnt = sum(car_list)
  #計算車位價格，車位價格須用總價減去建築物總價(坪數*單位價格)
  car_price = []
  for case in range(tot_case_cnt):
      if car_list[case] > 0:
          if df_filter['the berth total price Yuan'].iloc[case] > 0:
              car_price.append(df_filter['the berth total price Yuan'].iloc[case])
          else:
              car_price.append(df_filter['total price Yuan'].iloc[case] - 
             ((df_filter['building shifting total area'].iloc[case] - 
             df_filter['berth shifting total area square  meter'].iloc[case]) * 
             df_filter['the unit price a Yuan square meter'].iloc[case]))
  car_real_tot_price = sum(car_price)
  #計算每件案件平均價格
  tot_case_price_avg = tot_case_price / tot_case_cnt
  #計算車位平均價格
  car_real_tot_price_avg = car_real_tot_price / car_tot_cnt

    # In[8]:

    #count = pd.DataFrame({'總件數':tot_case_cnt, '總車位數':car_tot_cnt, '平均總價元':tot_case_price_avg, '平均車位總價元': car_real_tot_price_avg})
    count_data = [tot_case_cnt,car_tot_cnt,tot_case_price_avg,car_real_tot_price_avg]
    #將計算後的資料寫入CSV，為確保資料用excel打開中文編碼可以正確，編碼改用'utf-8-sig'帶入ＢOM   
    with open("count.csv","w",encoding='utf-8-sig') as csvfile:
        writer = csv.writer(csvfile)
        #先寫入columns_name
        writer.writerow(["總件數","總車位數","平均總價元","平均車位總價元"])
        writer.writerow(count_data)
~~~~
