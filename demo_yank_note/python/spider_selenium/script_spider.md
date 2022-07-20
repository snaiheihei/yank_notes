# script_spider

```py
import time
from driver import Driver
from selenium.webdriver.common.action_chains import ActionChains
import csv
import requests
import re

patient_infos = []
driver = Driver()

def logon(url_logon, name, passwd):
    driver.open_url(url_logon)
    driver.input_data("id", "account", name)
    driver.input_data("id", "password", passwd)
    driver.click("id", "login-button")
    print("="*50)
    print("logon successful~")
    driver.click("css", "#mCSB_4 > div.mCSB_container > div.ares3-sidebar-container.ng-isolate-scope > ul > li:nth-child(2) > a")
    driver.switch_iframe(driver.locateElement("css", "#hebe-container-level-0 > iframe"))
    print("switch iframe successful")
    # 跳过引导，切到今日
    driver.click("css", ".introjs-skipbutton")
    driver.click("css", "#switch-appt-date > button")
    time.sleep(5)

def get_eventID():
    # 获取所有event ID
    event_id = []
    els_eventID = driver.locateElements("css", "#calendar-container > div.dhx_cal_data div[event_id]")
    for el in  els_eventID:
        id = el.get_attribute("event_id")
        event_id.append(id)
    # 展开更多
    more_arrow = driver.locateElement("css", "#calendar-container > div.event-count-hint.next.ng-star-inserted > div.count")
    if more_arrow:
        # headless 模式下会出现click not reachable exception
        my_actionChain = ActionChains(driver.driver)
        my_actionChain.move_to_element(more_arrow).perform()
        driver.click("css", "#calendar-container > div.event-count-hint.next.ng-star-inserted > div.count")
        time.sleep(8)
        more_els_eventID = driver.locateElements("css", "#calendar-container > div.dhx_cal_data div[event_id]")
        for el in more_els_eventID:
            id = el.get_attribute("event_id")
            event_id.append(id)
    return list(set(event_id))

def get_patientID(eventIDs):
    # 请求API 获取所有的patient ID
    # 获取所有event_id , https://api-hn01.linkedcare.cn:9001/api/v1/appointment/1298361  >> 获取patient_id
    patientIds = []
    current_cookies = driver.driver.get_cookies()
    s = requests.Session()
    [s.cookies.set(c['name'], c['value']) for c in current_cookies]
    headers = {
        'authorization': 'bearer pV3sQFrJZEcKcyKRVMqoSLeZvieck9X4r3v9tdK9C8uBe-JLvA649DIRTDc5osQma6AgvJBDs-SFhwQfm_uFVJ_U1Zpi0QYynlNExCodbLuZw6F1mW1zQBouuz2xISH7gXXLNn85mSuRiDx9dJqw45Wt0xbrVzDXSmbFDT9ODsbIBL3qC9pN6-R7XlCMsSkPhLnC-FNQ2VXI5A4iG81J1qFMgbnEtZhIsjZoqXk-RKvEf_t2BokV3JR2lrYl8YMOcPHEBthWmQV2xDgAUWZweoU0SB4Mn4Yw5g7GGxx9mjM.eyJ0aWQiOiIxYjdiZTBjMy1lNTliLTRhZDctODcyNC03MDUxZWEwN2IxN2EifQ==',
        'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.5060.66 Safari/537.36',
    }
    for eid in eventIDs:
        url_patient_info = f"https://api-hn01.linkedcare.cn:9001/api/v1/appointment/{eid}"
        res = s.get(url_patient_info, headers=headers)
        print("patientID: ", res.json()["patientId"])
        patientIds.append(res.json()["patientId"])
    s.close()
    return  patientIds

def get_patientInfo(i, url_patient_detail):
    # 请求每一个patient 页面，获取详细信息
    # https://bjhzck.linkedcare.cn/ares3/#/patient/info/406275/record
    driver.driver.get(url_patient_detail)
    # 获取数据
    time.sleep(0.5)
    pt_name = driver.get_text("css", "td > label.ng-binding")
    pt_num = driver.get_text("css", "#patient-info-base-wrap > div.panel.panel-default > div.panel-body > div:nth-child(1) > div.sub-panel-body > table > tbody > tr:nth-child(2) > td:nth-child(1) > span.notes-value.ng-binding")
    time.sleep(0.3)
    driver.click("css", "#patientContainer > div.patient-content-container.k-pane > div:nth-child(1) > div:nth-child(2) > div.patient-menu-bar-view > div > ul > li:nth-child(2) > span")
    date = driver.get_text("css", "#timeline > div:nth-child(1) > div.timeline-icon.bg-primary.ng-binding > strong")
    diagnostic = driver.get_text("css", "#timeline > div:nth-child(1) > div.timeline-content > div.timeline-heading.clearfix.timeBox > table > tbody > tr > td:nth-child(1) > h3 > strong")
    dc_name = driver.get_text("css", "#timeline > div:nth-child(1) > div.timeline-content > div.timeline-body > ul > li:nth-child(2) > span:nth-child(1) > strong")
    project = driver.get_text("css", "#timeline > div:nth-child(1) > div.timeline-content > div.timeline-body > ul > li.ng-scope > span > strong")
    time.sleep(0.3)
    driver.click("css", "#patientContainer > div.patient-content-container.k-pane > div:nth-child(1) > div:nth-child(2) > div.patient-menu-bar-view > div > ul > li:nth-child(3) > span")
    order_cost = driver.get_text("css", "#chargeorderPaging > table > tbody > tr:nth-child(1) > td:nth-child(9)")
    note = driver.get_text("css", "#chargeorderPaging > table > tbody > tr:nth-child(1) > td:nth-child(12) > span")
    return [i, date, diagnostic, dc_name, pt_num, pt_name, project, order_cost, note]

regex = re.compile(".诊")
def clean_info(patientInfo):
    # [1, '7月17日', '14:30 - 16:00 复诊预约 预约 【合众齿科】', '朴珍荣(未知科室)', '22061215', '陈尚林', '[根管治疗]  ', '810.00', '']
    patientInfo[2] = regex.search(patientInfo[2]).group()
    patientInfo[3] = patientInfo[3].split("(")[0]
    print(patientInfo)
    return patientInfo

def writeCSV(data):
    header = ['index', 'date', 'diagnostic', 'dc_name', 'pt_num', 'pt_name', 'project', 'order_cost', 'note']
    with open("result.csv", "w", encoding="utf-8-sig", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(header)
        writer.writerows(data)

if __name__ == '__main__':
    url_logon = "https://bjhzck.linkedcare.cn/LogOn"
    name = "蔡有菊"
    passwd = "a123456"
    logon(url_logon, name, passwd)
    eventIDs = get_eventID()
    patientIDs = get_patientID(eventIDs)

    # 请求每一个patient 页面，获取详细信息
    i = 1
    for pid in patientIDs:
        url_patient_detail = f"https://bjhzck.linkedcare.cn/ares3/#/patient/info/{pid}/record"
        print(url_patient_detail)
        patientInfo = get_patientInfo(i, url_patient_detail)
        patientInfo_clean = clean_info(patientInfo)
        i += 1
        patient_infos.append(patientInfo_clean)
    # 保存数据到CSV
    writeCSV(patient_infos)
    print("main routine exit ...")

```

