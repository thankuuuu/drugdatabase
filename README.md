# drugdatabase

## Introduction

In this project, I scrape the data from https://www.webmd.com/ website which contains information about health including drug, medicine and doctor for finding the most effective drug from customer review information. You can find the recommended way to cure your illness from this website. But in this project, I will focus on drug information of common condition as the website provide customer to review and rate the drug. So I will scrape this information to see which drug has the most effective and satisfaction to customer. Moreover, I will take a look at the effective of each drug form and what form is commonly use. 
Enjoy the programing. 

## Part 1 : Scraping Data

The first process is to scrape the drug information for each condition from the website. The information would consist of drug name, indication, type, form, average price, number of review, effective, ease of use, satisfaction from customer review. I use Beautiful Soup to read the content of html and collect all the links for all common condition which have 37 conditions. After that, I use for loop with selenium to open all links to find all drug links for each condition and then collect all information. Finally, I delete all Null values in data frame and save it as csv file.

```python
from selenium import webdriver
from bs4 import BeautifulSoup
import requests
import re
import pandas as pd
import numpy as np
import time
```


```python
# Get all common condition from WebMD
web = requests.get('https://www.webmd.com/drugs/2/conditions/index')
condition = []
links = []
data = BeautifulSoup(web.content, 'lxml')
body = data.find('body')
main = body.find('main')
div = main.find("div",attrs={"class": "drugs-common-results"})
for link in div.find_all("a"):
    condition.append(link.text)
    links.append('https://www.webmd.com/'+link.get("href"))
```

```python
cond= []
review_url = []
drug_url = []
temp = []
i=0

# Collect all information from each condition links
for url in links :
    cond_temp = condition[i]
    try :
        # get cond,drug,indication, Tpye, reviews,review_url,drug_url
        web = requests.get(url)
        data = BeautifulSoup(web.content, 'lxml')
        body = data.find('body')
        main = body.find('main')
        div = main.find("table",attrs={"class": "drugs-treatments-table"})
        for link in div.find_all("a"):
            # collect review url
            if re.match("/drugs/drugreview", link.get('href')):
                review_url.append('https://www.webmd.com/'+link.get('href'))
                cond.append(cond_temp)
                
            # collect description url
            else :
                drug_url.append('https://www.webmd.com/'+link.get('href'))
                
        for link2 in div.find_all("td") :
            temp.append(link2.text)
    
        i+=1
    
    except :
        pass
```

```python
#temp contain drug name,indication,type and reviews. So we extract to each variables
indication = temp[1::4]
Type = temp[2::4]
reviews = temp[3::4]

#check the length of each list
print(len(cond))
print(len(indication))
print(len(Type))
print(len(reviews))
print(len(review_url))
print(len(drug_url))
```

```python
# search for drug that have been rated
index = []
for i in range (len(reviews)):
    if reviews[i] != '0 Reviews':
        index.append(i)
len(index)
```

```python
cond_use= []
indication_use = []
Type_use = []
reviews_use = []
review_url_use = []
drug_url_use = []

for ind in index :
    cond_use.append(cond[ind])
    indication_use.append(indication[ind])
    Type_use.append(Type[ind])
    reviews_use.append(reviews[ind])
    review_url_use.append(review_url[ind])
    drug_url_use.append(drug_url[ind])
```

```python
#get drug name & price link
drug = []
links2 = []
for url in drug_url_use :
    try :
        web = requests.get(url)
        data = BeautifulSoup(web.content, 'lxml')
        body = data.find('body')
        div = body.find("div",attrs={"class": "drug-names"})
        for name in div.find_all("p"):
            if 'GENERIC' in name.text:
                drug.append(name.text[17:])
                
        div2 = div.find("a",attrs={"class": "drug-lowest-btn"})
        links2.append(div2.get('href'))
    
    except :
        drug.append(None)
        links2.append(None)
```

```python
#Get information of drug
infor = []
driver = webdriver.Chrome()

for url in links2 :
    try :
        driver.get(url)
        time.sleep(1)
        page = driver.page_source
        data = BeautifulSoup(page, 'lxml')
        body = data.find('body')
        div = body.find("div",attrs={"id": "main-section"})
        div2 = div.find("p",attrs={"class": "drug-desc"})
        infor.append(div2.text)

    except :
        infor.append(None)
```

```python
i = 0
for x in infor:
    if x is None:
        i+=1
i
```

```python
#Get effectiveness, ease of use and satisfaction from review_url
effective = []
ease_use = []
satisfaction = []
driver = webdriver.Chrome()

for url in review_url_use :
    try :
        driver.get(url)
        page = driver.page_source
        data = BeautifulSoup(page, 'lxml')
        body = data.find('body')
        div = body.find("div",attrs={"id": "centering_area"})
        eff = div.find("p",attrs={"id":"EffectivenessSummaryValue"})
        easeuse = div.find("p",attrs={"id":"EaseOfUseSummaryValue"})
        satis = div.find("p",attrs={"id":"SideEffectsSummaryValue"})
        effective.append(eff.text.replace('(','').replace(')',''))
        ease_use.append(easeuse.text.replace('(','').replace(')',''))
        satisfaction.append(satis.text.replace('(','').replace(')',''))

    except :
        effective.append(None)
        ease_use.append(None)
        satisfaction.append(None)

```

```python
# Create data frame
join = {'Condition':cond_use,'Drug':drug,'Indication':indication_use,'Type':Type_use,'Reviews':reviews_use,'Effective':effective,'EaseOfUse':ease_use,'Satisfaction':satisfaction,'Information':infor}
df = pd.DataFrame(join)
```

```python
#Clean all None value
df2 = df.dropna(how='any',axis=0)
```

```python
# Export as csv file
df2.to_csv('Drug.csv',index =False)
```
## Part 2 : Cleaning Data

The next process is cleaning and preparing data for further analysis.  As the website did not contain the specific part of average price and form of the drug, I can take just paragraph of information. I have to extract information from the paragraph, I use tokenize function to get the sentence that contain ‘average’ word which contain average price and form in the sentence. After that, I split the sentence into word and get the information from it. The last cleaning process is to sum all drug review rating and find the average of drug that have the same name, same form for same condition as sometime the website contain the duplicate drug information. Finally, I have 685 drugs left for 37 conditions.

### 2.1 Cleaning Process 


```python
import pandas as pd
import numpy as np
from nltk import tokenize
```


```python
Drug_clean = pd.read_csv('Drug.csv')
Drug_clean.head(10)
```

```python
# Remove 'Reviews' word from the Reviews column
for i in range (Drug_clean.shape[0]):
    Drug_clean.loc[i,'Reviews'] = Drug_clean.loc[i,'Reviews'][0:len(Drug_clean.loc[i,'Reviews'])-8]
```

## 2.2 Preparing Process 

### Price 

```python
# Get Price from information
Price = []
for i in range (Drug_clean.shape[0]):
    para = Drug_clean.iloc[i,8]
    sent = tokenize.sent_tokenize(para)
    for x in sent :
        if 'average' in x :
            words = x.split()
    for x in words :
        if '$' in x:
            z = x.replace("$", "")
            z = z[0:len(z)-1]
            Price.append(z)
```

### Form of drug 

```python
# Try to find all form of drug
count = []
all_form = ['tablets','Capsule(s)','Capsule','Tablet(s)','Tablet','tablets','Bottle','Vial(s)','Vial','Reconstituted(s)','Reconstituted','Tube','Jar','Can','Box','Syringe']
for i in range (Drug_clean.shape[0]):
    para = Drug_clean.iloc[i,8]
    sent = tokenize.sent_tokenize(para)
    for x in sent :
        if 'average' in x :
            words = x.split()
    for word in words:
        word = word.replace(",", "")
        # count collect all index that can collect form of drug as we know in all_form variables
        if word in all_form :
            count.append(i)
            break
```

```python
# Find number of index that didn't collect the form of drug and find the new form
for i in range (Drug_clean.shape[0]):
    if i not in count:
        print(i)
```
```python
#Use index from check and check the sentence that didn't collect the form of frug
i = 667
para = Drug_clean.iloc[i,8]
sent = tokenize.sent_tokenize(para)
for x in sent :
    if 'average' in x :
        words = x.split()
        for word in words:
            word = word.replace(",", "")
            print(word)
```

```python
Drug_clean.iloc[667]
```

```python
# Get Form of Drug from information

#Get all type of form by above process
all_form = ['Capsule(s)','Capsule','Tablet(s)','Tablet','tablets','Bottle','Vial(s)','Vial','Reconstituted(s)','Reconstituted','Tube','Jar','Can','Box','Syringe','Implant','Package','Pen(s)','Inhaler']
Form = []
for i in range (Drug_clean.shape[0]):
    para = Drug_clean.iloc[i,8]
    sent = tokenize.sent_tokenize(para)
    for x in sent :
        if 'average' in x :
            words = x.split()
    for word in words:
        word = word.replace(",", "")
        if word in all_form :
            Form.append(word)
            break
```

```python
### count frequency of each form
from collections import Counter
counts = Counter(Form)
```

```python
# 
tablet = ['Tablet(s)','Tablet','tablets']
capsule = ['Capsule(s)','Capsule']
cream = ['Tube','Can','Jar']
liquid_drink = ['Bottle']
liquid_inject = ['Vial','Reconstituted','Reconstituted(s)','Vial(s)','Pen(s)','Syringe']
other = ['Box','Package','Implant','Inhaler']
for i in range (len(Form)):
    if Form[i] in tablet:
        Form[i] = 'Tablet'
    elif Form[i] in capsule:
        Form[i] = 'Capsule'
    elif Form[i] in cream:
        Form[i] = 'Cream'
    elif Form[i] in liquid_drink :
        Form[i] = 'Liquid (Drink)'
    elif Form[i] in liquid_inject :
        Form[i] = 'Liquid (Inject)'
    elif Form[i] in other:
        Form[i] = 'Other'
```


## 2.3 Concatenating 


```python
# Deleting Informtion columns
Drug_clean = Drug_clean.drop(['Information'], axis=1)
Drug_clean.head(10)
```

```python
join = {'Form':Form,'Price':Price}
df2 = pd.DataFrame(join)
df2.shape
```

```python
Data = pd.concat([Drug_clean,df2], axis=1)
Data.head(10)
```

```python
#Change String to Float
Data['Reviews'] = Data['Reviews'].astype(float)
Data['Effective'] = Data['Effective'].astype(float)
Data['EaseOfUse'] = Data['EaseOfUse'].astype(float)
Data['Satisfaction'] = Data['Satisfaction'].astype(float)
Data['Price'] = Data['Price'].astype(float)
```

```python
#Delete duplicates data
df = pd.pivot_table(Data, index=['Condition','Drug','Indication','Type','Form'],aggfunc='mean')
Data2 = pd.DataFrame()
for i in range (len(df.index)):
        temp = {'Condition': df.index[i][0],'Drug':df.index[i][1],'Indication':df.index[i][2],'Type':df.index[i][3],'Form':df.index[i][4],'EaseOfUse':df.iloc[i,0],'Effective':df.iloc[i,1] ,'Price':df.iloc[i,2],'Reviews':df.iloc[i,3] ,'Satisfaction':df.iloc[i,4]} 
        Data2 = Data2.append(temp, ignore_index=True)
Data2
```

```python
# Export as csv file
Data2.to_csv('Drug_clean.csv', index = False)
```
    
# Part 3 : Drug Database

As I would like to see the effective of drug by form, indication, type and condition, so I sum all data up and find the average by each group. After I got all the data, I create the drug Database. The drug database consists of 7 function.
First function, you can see the top drug for each condition by specific criteria.
Next function is to see the overall drug of each condition by criteria. 
Third function, it will show the boxplot of different form of drug for specific condition by criteria.
The next one is nearly the same as the third function, show overall rating the boxplot of all drug in database by different form.
The next two function is to see the different overall rating of OnLabel and OffLable, RX and OTC.
The last function is to see the what is the most commonly form of drug in the market by pie chart.

```python
import pandas as pd
import numpy as np
import seaborn as sns
from matplotlib import pyplot as plt
```


```python
Drug = pd.read_csv('Drug_clean.csv')
for i in range (Drug.shape[0]) :
    Drug.loc[i,'Condition'] = Drug.loc[i,'Condition'].capitalize()
Drug.head(10)
```

### Preparing Data

```python
#Change String to Float
Drug['Reviews'] = Drug['Reviews'].astype(float)
Drug['Effective'] = Drug['Effective'].astype(float)
Drug['EaseOfUse'] = Drug['EaseOfUse'].astype(float)
Drug['Satisfaction'] = Drug['Satisfaction'].astype(float)
Drug['Price'] = Drug['Price'].astype(float)
```

### Form of Drug

```python
#Subsetting each form of drug
Tablet = Drug.loc[(Drug.Form == 'Tablet'),]
Capsule = Drug.loc[(Drug.Form == 'Capsule'),]
Cream = Drug.loc[(Drug.Form == 'Cream'),]
Liquid_Drink = Drug.loc[(Drug.Form == 'Liquid (Drink)'),]
Liquid_Inject = Drug.loc[(Drug.Form == 'Liquid (Inject)'),]
Other = Drug.loc[(Drug.Form == 'Other'),]
```

```python
Form_sum = pd.DataFrame()
Form = [Tablet,Capsule,Cream,Liquid_Drink,Liquid_Inject,Other]
Form_name = ['Tablet','Capsule','Cream','Liquid_Drink','Liquid_Inject','Other']
i = 0
for form in Form :
    sum_temp = form.sum(axis = 0, skipna = True)
    temp = {'Form': Form_name[i],'Reviews_avg':sum_temp[7]/form.shape[0],'Effective_avg':sum_temp[3]/form.shape[0],'EaseOfUse_avg':sum_temp[2]/form.shape[0],'Satisfaction_avg':sum_temp[8]/form.shape[0],'Price_avg':sum_temp[6]/form.shape[0]}
    Form_sum = Form_sum.append(temp, ignore_index=True)
    i += 1
```

```python
Form_sum = Form_sum.set_index('Form')
Form_sum
```

### Indication

```python
#Subsetting each indication
On_Label = Drug.loc[(Drug.Indication == 'On Label'),]
Off_Label = Drug.loc[(Drug.Indication == 'Off Label'),]
```

```python
Indication_sum = pd.DataFrame()
Indication = [On_Label,Off_Label]
Indication_name = ['On Label','Off Label']
i = 0
for indic in Indication :
    sum_temp = indic.sum(axis = 0, skipna = True)
    temp = {'Indication': Indication_name[i],'Reviews_avg':sum_temp[7]/indic.shape[0],'Effective_avg':sum_temp[3]/indic.shape[0],'EaseOfUse_avg':sum_temp[2]/indic.shape[0],'Satisfaction_avg':sum_temp[8]/indic.shape[0],'Price_avg':sum_temp[6]/indic.shape[0]}
    Indication_sum = Indication_sum.append(temp, ignore_index=True)
    i += 1
```

```python
Indication_sum = Indication_sum.set_index('Indication')
Indication_sum
```

### Type

```python
#Subsetting each indication
RX = Drug.loc[(Drug.Type == 'RX'),]
OTC = Drug.loc[(Drug.Type == 'OTC'),]
RX_OTC = Drug.loc[(Drug.Type == 'RX/OTC'),]
```

```python
Type_sum = pd.DataFrame()
Type = [RX,OTC,RX_OTC]
Type_name = ['RX','OTC','RX/OTC']
i = 0
for typ in Type :
    sum_temp = typ.sum(axis = 0, skipna = True)
    temp = {'Type': Type_name[i],'Reviews_avg':sum_temp[7]/typ.shape[0],'Effective_avg':sum_temp[3]/typ.shape[0],'EaseOfUse_avg':sum_temp[2]/typ.shape[0],'Satisfaction_avg':sum_temp[8]/typ.shape[0],'Price_avg':sum_temp[6]/typ.shape[0]}
    Type_sum = Type_sum.append(temp, ignore_index=True)
    i += 1
```

```python
Type_sum = Type_sum.set_index('Type')
Type_sum
```

### Condition


```python
Data_grouped_condition = Drug.groupby(['Condition'])
Data_grouped_condition.describe().head()
```
```python
# Collect the averge of Reviews, Effective, Ease of Use, Satisfaction , Price
Condition_sum = Data_grouped_condition.mean()
```

```python
# Making a dictionay of condition
Condition_grouped = list(Data_grouped_condition)
Condition_dict = { Condition_sum.index[i] : i for i in range(0, len(Condition_sum.index) ) }
```

```python
def menu():
    print()
    print('1. Top Drug for each condition')
    print('2. Overall Drug of each condition')
    print('3. Best Form of Drug for each condition')
    print('4. Best Form of Drug')
    print('5. On Label vs Off Label Drug')
    print('6. RX vs OTC')
    print('7. Form of Drug')
    print('8. Condition name sheet')
    print('q. Quit the program')
    print()
    print('Please type "8" for Condition input')
    print()

def processRequest(selection):
    if selection == '1':
        TopDrug(selection)
    elif selection == '2':
        Overall(selection)    
    elif selection == '3':
        BestFormCondition(selection) 
    elif selection == '4':
        BestForm(selection)
    elif selection == '5':
        OnvsOff(selection)
    elif selection == '6':
        RXvsOTC(selection)
    elif selection == '7':
        DrugForm(selection)  
    elif selection == '8':
        ConditionNameSheet(selection) 
    else:
        return 'q' 

def TopDrug(selection):
    print()
    print('Input the Condition')
    print('For Condition name guideline, please selection "8" on the main menu')
    print()
    Condition = input('Enter Condition : ').capitalize().strip()
    print()
    print('Input the Criteria')
    print('Critiria : Effective, Ease of Use, Satisfaction, Price')
    print()    
    Criteria = input('Enter Criteria : ').lower().replace(" ","")
    print('Input the Minimum of Reviews')
    print()    
    number = input('Enter Number : ').replace(" ","")
        
    try :
        Condition_data = Condition_grouped[Condition_dict[Condition]]
        if Criteria == 'effective':
            Top_Drug = Condition_data[1].sort_values(by='Effective', ascending=False)
        if Criteria == 'easeofuse':
            Top_Drug = Condition_data[1].sort_values(by='EaseOfUse', ascending=False)
        if Criteria == 'satisfaction':
            Top_Drug = Condition_data[1].sort_values(by='Satisfaction', ascending=False)
        if Criteria == 'price':
            Top_Drug = Condition_data[1].sort_values(by='Price')
        Top_Drug = Top_Drug.set_index('Condition')
        Top_Drug = Top_Drug[Top_Drug['Reviews'] > int(number)]
        print('Top ' + number + ' Drug for ' + Condition)
        print(Top_Drug.head(5))
            
            
    except :
        print()
        print('Error : Typing Wrong Input')
                       
def Overall(selection):
    print()
    print('Input the Criteria')
    print('Critiria : Effective, Ease of Use, Satisfaction, Price')
    print()    
    Criteria = input('Enter Criteria : ').lower().replace(" ","")
        
    try :
        Top_Drug = Drug.copy()
        if Criteria == 'effective':
            Cri = 'Effective'
        if Criteria == 'easeofuse':
            Cri = 'EaseOfUse'
        if Criteria == 'satisfaction':
            Cri = 'Satisfaction'
        if Criteria == 'price':
            Cri = 'Price'
            # Some Drug are really expensive, so will show just all drugs that less than 500$
            Top_Drug = Top_Drug[Top_Drug['Price'] < 200]
        plt.figure(figsize=(24,12))
        Plot = sns.boxplot(x="Condition", y=Cri, data=Top_Drug)
        Plot.axes.set_title("Overall Drug by "+Cri,fontsize=30)
        Plot.set_xlabel('Condition',fontsize=20)
        Plot.set_ylabel(Cri,fontsize=20)
        Plot.tick_params(labelsize=15)
        Plot.set_xticklabels(Plot.get_xticklabels(),rotation=90)
        plt.show()
        
    except :
        print()
        print('Error : Typing Wrong Input')
    
    
def BestFormCondition(selection):
    print()
    print('Input the Condition')
    print('For Condition name guideline, please selection "8" on the main menu')
    print()
    Condition = input('Enter Condition : ').capitalize().strip()
    print()
    print('Input the Criteria')
    print('Critiria : Effective, Ease of Use, Satisfaction, Price')
    print()    
    Criteria = input('Enter Criteria : ').lower().replace(" ","")
        
    try :
        Condition_data = Condition_grouped[Condition_dict[Condition]][1]
        if Criteria == 'effective':
            Cri = 'Effective'
        if Criteria == 'easeofuse':
            Cri = 'EaseOfUse'
        if Criteria == 'satisfaction':
            Cri = 'Satisfaction'
        if Criteria == 'price':
            Cri = 'Price'
            # Some Drug are really expensive, so will show just all drugs that less than 500$
            Condition_data = Condition_data[Condition_data['Price'] < 200]
        plt.figure(figsize=(24,12))
        Plot = sns.boxplot(x="Form", y=Cri, data=Condition_data)
        Plot.axes.set_title(Cri + ' of ' + Condition+ "'s Drug",fontsize=30)
        Plot.set_xlabel('Form',fontsize=20)
        Plot.set_ylabel(Cri,fontsize=20)
        Plot.tick_params(labelsize=15)
        Plot.set_xticklabels(Plot.get_xticklabels(),rotation=90)
        plt.show()
            
    except :
        print()
        print('Error : Typing Wrong Input')
        
def BestForm(selection):
    print()
    print('Input the Criteria')
    print('Critiria : Effective, Ease of Use, Satisfaction, Price')
    print()    
    Criteria = input('Enter Criteria : ').lower().replace(" ","")
        
    try :
        Top_Drug = Drug.set_index('Condition')
        if Criteria == 'effective':
            Cri = 'Effective'
        if Criteria == 'easeofuse':
            Cri = 'EaseOfUse'
        if Criteria == 'satisfaction':
            Cri = 'Satisfaction'
        if Criteria == 'price':
            Cri = 'Price'
            # Some Drug are really expensive, so will show just all drugs that less than 500$
            Top_Drug = Top_Drug[Top_Drug['Price'] < 200]
        plt.figure(figsize=(24,12))
        Plot = sns.boxplot(x="Form", y=Cri, data=Top_Drug)
        Plot.axes.set_title("Best Form by "+Cri,fontsize=30)
        Plot.set_xlabel('Form',fontsize=20)
        Plot.set_ylabel(Cri,fontsize=20)
        Plot.tick_params(labelsize=15)
        Plot.set_xticklabels(Plot.get_xticklabels(),rotation=90)
        plt.show()
        print(Form_sum)

    except :
        print()
        print('Error : Typing Wrong Input')

def OnvsOff(selection) :
    print()
    print('On Label medications are FDA approved for the treatment of this condition')
    print('Off Label medications are occasionally used but are not FDA approved for the treatment of this condition')
    print()
    print('Input the Criteria')
    print('Critiria : Effective, Ease of Use, Satisfaction, Price')
    print()    
    Criteria = input('Enter Criteria : ').lower().replace(" ","")
        
    try :
        On_Drug = Drug[Drug['Indication'] == 'On Label']
        Off_Drug = Drug[Drug['Indication'] == 'Off Label']
        Top_Drug = pd.concat([On_Drug,Off_Drug], axis=0)
        if Criteria == 'effective':
            Cri = 'Effective'
        if Criteria == 'easeofuse':
            Cri = 'EaseOfUse'
        if Criteria == 'satisfaction':
            Cri = 'Satisfaction'
        if Criteria == 'price':
            Cri = 'Price'
            # Some Drug are really expensive, so will show just all drugs that less than 500$
            Top_Drug = Top_Drug[Top_Drug['Price'] < 200]
        plt.figure(figsize=(24,12))
        Plot = sns.boxplot(x="Indication", y=Cri, data=Top_Drug)
        Plot.axes.set_title("Indication by "+Cri,fontsize=30)
        Plot.set_xlabel('Indication',fontsize=20)
        Plot.set_ylabel(Cri,fontsize=20)
        Plot.tick_params(labelsize=15)
        Plot.set_xticklabels(Plot.get_xticklabels(),rotation=90)
        plt.show()
        print(Indication_sum)
    
    except :
        print()
        print('Error : Typing Wrong Input')
        
def RXvsOTC(selection) :
    print()
    print('Rx medications require a prescription')
    print('OTC medications may be purchased Over the Counter')
    print()
    print('Input the Criteria')
    print('Critiria : Effective, Ease of Use, Satisfaction, Price')
    print()    
    Criteria = input('Enter Criteria : ').lower().replace(" ","")
        
    try :
        RX_Drug = Drug[Drug['Type'] == 'RX']
        OTC_Drug = Drug[Drug['Type'] == 'OTC']
        RXOTC_Drug = Drug[Drug['Type'] == 'RX/OTC']
        Top_Drug = pd.concat([RX_Drug,OTC_Drug,RXOTC_Drug], axis=0)
        if Criteria == 'effective':
            Cri = 'Effective'
        if Criteria == 'easeofuse':
            Cri = 'EaseOfUse'
        if Criteria == 'satisfaction':
            Cri = 'Satisfaction'
        if Criteria == 'price':
            Cri = 'Price'
            # Some Drug are really expensive, so will show just all drugs that less than 500$
            Top_Drug = Top_Drug[Top_Drug['Price'] < 200]
        plt.figure(figsize=(24,12))
        Plot = sns.boxplot(x="Type", y=Cri, data=Top_Drug)
        Plot.axes.set_title("Type by "+Cri,fontsize=30)
        Plot.set_xlabel('Type',fontsize=20)
        Plot.set_ylabel(Cri,fontsize=20)
        Plot.tick_params(labelsize=15)
        Plot.set_xticklabels(Plot.get_xticklabels(),rotation=90)
        plt.show()
        print(Type_sum)
    
    except :
        print()
        print('Error : Typing Wrong Input')
        
def DrugForm(selection) :
    explode = (0.05,0.05,0.05,0.05,0.05,0.05)
    colors = ['#ff9999','#66b3ff','#99ff99','#ffcc99','#fed0fc','#ceaefa']

    Data_group = Drug.groupby(['Form'])
    Data_size = Data_group.size()
    Data_size = pd.DataFrame(Data_size)
    Data_size.columns = ['Total']
    DrugOrder = Data_size.sort_values(by='Total', ascending = False)
    plt.pie(DrugOrder, labels=DrugOrder.index, explode = explode, autopct='%1.1f%%', colors = colors)
    plt.title('Form of Drug')
    plt.legend(loc='best', bbox_to_anchor=(1, 0.5))
    plt.show()
        
def ConditionNameSheet(selection) :
    conditionnamesheet = pd.DataFrame(Condition_sum.index)
    print(conditionnamesheet)

def main():
    print('Welcome to Common Drugs Database')     
    selection = ''
    while selection != 'q':
        menu()
        selection = input('Make a selection: ')
        response = processRequest(selection)
        if response == 'q':
            break
                                 
main()
```
    
## Conclusion :

In conclusion, you can see from the pie chart, tablet is the most commonly sale in the market for relief 37 common condition. Next we take a look at price and effective of each form, we can see the effective rating are nearly the same, but liquid drink form is the most cheapest. So if you can, you would consider to buy liquid drug for saving your money. This boxplot show that price and effective of drug are not in the same trend which mean that expensive drug doesn’t mean the effective one. For the obvious example, meniere’s drug is really cheap but the effective rate is relatively high. The next graph btw on and off label, On-Label medications - FDA approved for the treatment of this condition. Off-Label medications - occasionally used but are not FDA approved for the treatment of this condition. Ease of use and effective average seem to be the same, but on-label drug are cheaper. For RX and OTC, Rx medications - require a prescription. OTC medications - may be purchased over the Counter. The average price of OTC drug are relative low compare to RX drug, and the effective of OTC drug is higher than RX. Moreover, OTC is easiler to find in the market, so you would like to buy this type of drug when you have some illness.
    


