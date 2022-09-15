---
layout: post
title: "pyspider爬取并解析网页中table数据的例子"
keywords: ["python"]
description: "python"
category: "python"
tags: ["python","python"]
---

主要爬取一些第三方接口参数，字段，说明信息，有很多个下面这种类似的页面，这些都是table数据
```
<table class="confluenceTable">
    <colgroup>
     <col style="width: 130.0px;" />
     <col style="width: 77.0px;" />
     <col style="width: 384.0px;" />
    </colgroup>
    <tbody>
     <tr>
      <th class="confluenceTh">参数</th>
      <th class="confluenceTh">是否必填</th>
      <th class="confluenceTh">说明</th>
     </tr>
     <tr>
      <td class="confluenceTd">appid</td>
      <td class="confluenceTd">Y</td>
      <td class="confluenceTd"><br /></td>
     </tr>
     <tr>
      <td class="confluenceTd"><span>idNo</span></td>
      <td class="confluenceTd">Y</td>
      <td class="confluenceTd"><span>证件号码</span></td>
     </tr>
     <tr>
      <td class="confluenceTd"><span>name</span></td>
      <td class="confluenceTd"><span>Y</span></td>
      <td class="confluenceTd"><span>姓名</span></td>
     </tr>
     <tr>
      <td colspan="1" class="confluenceTd"><span>photoStr</span></td>
      <td colspan="1" class="confluenceTd"><span>Y</span></td>
      <td colspan="1" class="confluenceTd"><span>照片文件</span><br /><span>注意：原始图片不能超过 2M，且必须为 JPG 或 PNG 格式</span></td>
     </tr>
     <tr>
      <td colspan="1" class="confluenceTd"><span>sourcePhotoStr</span></td>
      <td colspan="1" class="confluenceTd"><span>Y</span></td>
      <td colspan="1" class="confluenceTd"><span>合作伙伴自己提供的可信比对源照片</span><br /><span>注意：原始图片不能超过 2M，且必须为 JPG 或 PNG 格式</span></td>
     </tr>
     <tr>
      <td colspan="1" class="confluenceTd"><span>sourcePhotoType</span></td>
      <td colspan="1" class="confluenceTd"><span>Y</span></td>
      <td colspan="1" class="confluenceTd"><span>比对源照片类型</span><br /><span>1：网纹照</span><br /><span>2：高清照</span></td>
     </tr>
     <tr>
      <td colspan="1" class="confluenceTd">transationid</td>
      <td colspan="1" class="confluenceTd">Y</td>
      <td colspan="1" class="confluenceTd">每次请求唯一的标识（和第三方接口中的orderno对应）</td>
     </tr>
    </tbody>
   </table>
```

  原本用pyspider的response应该就可以了，response.doc返回的本来就是PyQuery，于是遍历所有tr,然后遍历tr内的td获取text就好了，     
  发现 1)有的td内部有span标签的会获取不到数据,2) 有很多td的text本来就是空的，二期也没有表头，不能直接用doc获取整个页面的td计算，
  也想过直接用etree,lxml.html   
没招了，只好试一试BeautifulSoup

详细代码

```
from bs4 import BeautifulSoup
from lxml.html import etree
import lxml.html
for item in response.doc('.confluenceTable').items():
    table_header_dict={}
    root = lxml.html.fromstring(item.html())
    soup = BeautifulSoup(item.html())
    for row in soup.findAll("tr"):
        # 获取tr内的th,得到表头信息
        th_cells = row.find_all("th")
        if len(th_cells)>0 :
            table_header_dict = self.get_table_header_dict(th_cells)
    #解析tr内的td,解析表格数据
    trs = soup.findAll("tr");
    td_dict_data = self.parse_tddata_to_dict(table_header_dict,trs)        
```
# 获取tr内的th,得到表头信息

```
#获取表头信息
def get_table_header_dict(self,th_cells):
    th_count = len(th_cells)
    #print th_count
    th_dict={}
    index =0
    for i in range(th_count):
        temp = th_cells[i].text
        if ((temp =='参数')|(temp =='参数名')|(temp =='参数名称')|(temp =='名称')|(temp =='字段')|(temp =='字段名')|(temp='字段名称')):
            th_dict[i]= 'name'
        elif ('类型' in temp ):
            th_dict[i]= 'type'   
        elif ((temp=='说明')|(temp=='备注')|(temp='参数说明')|(temp='字段说明')):
            th_dict[i]= 'desc'
        else :
            th_dict[i]= temp
    return th_dict
```

#解析tr内的td,解析表格数据
```
def parse_tddata_to_dict(self,table_header_dict,trs):
    ret_list = []
    retuslt = {}
    for row in trs:
            # 获取表格内的所有td
            cells = row.findAll("td")
            if len(cells)>0 :
                index = 0
                for i in range(len(cells)):
                       #直接通过text得到文本内容
                        print cells[index].text
                        retuslt[table_header_dict.get(index)] = cells[index].text
                        index = index+1

                ret_list.append(retuslt)
    #print len(ret_list)
    #print ret_list
    return ret_list;
```

直接用PyQuery获取不到不知道为啥，

```
for item in response.doc('.confluenceTable').items():
for i in range(len(item('td'))):
        field_name = th_dict[i%len(th_dict)]
       # print i,th_count,th_dict[i%th_count]
       #这个 item('td')[i].text很多时候(内部有 <span></span>标签)获取不到数据
        if item('td')[i] is not None and item('td')[i].text is not None:

            index = i%len(th_dict);
            field_name = th_dict[index]
            if 'name' in field_name or 'type' in field_name or 'desc' in field_name:
                td_data[field_name]=item('td')[i].text.encode("utf-8")
        else:
            print th_dict[i%len(th_dict)] +' no data'
            if 'name' in field_name or 'type' in field_name or 'desc' in field_name:
                field_name = th_dict[i%len(th_dict)]
                td_data[field_name]='null'

        if i>0 and i%len(th_dict) ==0:
            print td_data
            td_data_list.append(td_data)
            td_data = {}
            td_data['url'] =url
            td_data['title'] =title



    print td_data_list
```
用lxml.html应该也可以的

```
for item in response.doc('.confluenceTable').items():
    root = lxml.html.fromstring(item.html())
    print "th  header"
    header_data = root.xpath('//tr/th//text()')
    print '======================='
    print 'len(header_data):'
    print len(header_data)
    print "td data"
    td_data = root.xpath('//tr/td//text()')
    '''
    for row in root.xpath('//tr/td//text()'):
        print([i for i in row.itertext()])
    '''


    print 'len(td_data):'
    print len(td_data)
    print  '======================='
```
最后入库数据，大概是这个样子

```
*************************** 19. row ***************************
   fid: 3052
  furl: http://abc.xyz.com/pages/viewpage.action?pageId=24841569
ftitle: 【20180903】腾讯云-照片比对接口对接 - 风控业务需求 - Confluence
 fname: sourcePhotoType
 ftype: String
 fdesc: 比对源照片类型1：网纹照2：高清照
*************************** 20. row ***************************
   fid: 3053
  furl: http://abc.xyz.com/pages/viewpage.action?pageId=24841569
ftitle: 【20180903】腾讯云-照片比对接口对接 - 风控业务需求 - Confluence
 fname: sourcePhotoType
 ftype: String
 fdesc: 比对源照片类型1：网纹照2：高清照
20 rows in set (0.01 sec)
```

# 获取嵌套table的数据

```
<div class="table-wrap">
 <table class="wrapped confluenceTable">
  <colgroup>
   <col />
   <col />
   <col />
   <col />
  </colgroup>
  <tbody>
   <tr>
    <th class="confluenceTh">序号</th>
    <th class="confluenceTh">服务</th>
    <th class="confluenceTh">说明</th>
    <th colspan="1" class="confluenceTh">服务类型</th>
   </tr>
   <tr>
    <td class="confluenceTd">01</td>
    <td class="confluenceTd">查询接口</td>
    <td class="confluenceTd">征信中心封装查询接口给前端请求同盾H5链接</td>
    <td rowspan="2" class="confluenceTd">查询接口</td>
   </tr>
   <tr>
    <td class="confluenceTd">02</td>
    <td class="confluenceTd">查询接口</td>
    <td class="confluenceTd">征信中心将拿到的H5链接返回给前端</td>
   </tr>
   <tr>
    <td class="confluenceTd">03</td>
    <td class="confluenceTd">接收同盾登陆成功的回调</td>
    <td class="confluenceTd"><br /></td>
    <td colspan="1" class="confluenceTd">回调接口</td>
   </tr>
   <tr>
    <td colspan="1" class="confluenceTd">04</td>
    <td colspan="1" class="confluenceTd">接收同盾抓取原始数据成功的回调</td>
    <td colspan="1" class="confluenceTd"><p><br /></p></td>
    <td colspan="1" class="confluenceTd">回调接口</td>
   </tr>
   <tr>
    <td colspan="1" class="confluenceTd">05</td>
    <td colspan="1" class="confluenceTd">请求数据</td>
    <td colspan="1" class="confluenceTd">主动请求同盾的接口获取数据</td>
    <td colspan="1" class="confluenceTd"><br /></td>
   </tr>
   <tr>
    <td colspan="1" class="confluenceTd">06</td>
    <td colspan="1" class="confluenceTd">数据逻辑</td>
    <td colspan="1" class="confluenceTd">落mongo</td>
    <td colspan="1" class="confluenceTd"><br /></td>
   </tr>
   <tr>
    <td colspan="1" class="confluenceTd">07</td>
    <td colspan="1" class="confluenceTd">查询接口</td>
    <td colspan="1" class="confluenceTd">封装查询接口供请求方获取taskid</td>
    <td colspan="1" class="confluenceTd">查询接口</td>
   </tr>
   <tr>
    <td colspan="1" class="confluenceTd">08</td>
    <td colspan="1" class="confluenceTd">查询接口</td>
    <td colspan="1" class="confluenceTd"><span>封装查询接口供请求方获取运营商数据或中间状态</span></td>
    <td colspan="1" class="confluenceTd"><br /></td>
   </tr>
  </tbody>
 </table>
</div>
<div class="table-wrap">
 <table class="wrapped confluenceTable">
  <colgroup>
   <col style="width: 94.0px;" />
   <col style="width: 77.0px;" />
   <col style="width: 119.0px;" />
  </colgroup>
  <tbody>
   <tr>
    <td class="highlight-grey confluenceTd" data-highlight-colour="grey"><p>参数</p></td>
    <td class="highlight-grey confluenceTd" colspan="1" data-highlight-colour="grey">是否必传</td>
    <td class="highlight-grey confluenceTd" data-highlight-colour="grey"><p>含义</p></td>
   </tr>
   <tr>
    <td class="confluenceTd"><p>appid</p></td>
    <td colspan="1" class="confluenceTd"><span>Y</span></td>
    <td class="confluenceTd"><p>业务线Spuid</p></td>
   </tr>
   <tr>
    <td class="confluenceTd"><p>taskId</p></td>
    <td colspan="1" class="confluenceTd"><span>Y</span></td>
    <td class="confluenceTd"><p>任务id</p></td>
   </tr>
   <tr>
    <td colspan="1" class="confluenceTd"><span>transationId</span></td>
    <td colspan="1" class="confluenceTd"><span>Y</span></td>
    <td colspan="1" class="confluenceTd"><p>请求的唯一标识</p></td>
   </tr>
  </tbody>
 </table>
</div>
<h3 class="auto-cursor-target" id="id-【20180830】同盾运营商数据对接-回调通知">回调通知</h3>
<p>我们可以截取用户提交成功、原始报告获取结果（成功、失败、超时），魔方报告（成功、失败）。</p>

<div class="table-wrap">
 <table class="confluenceTable">
  <colgroup>
   <col />
   <col />
   <col />
  </colgroup>
  <tbody>
   <tr>
    <th class="confluenceTh">字段名称</th>
    <th colspan="1" class="confluenceTh"><span style="color: rgb(51,51,51);text-decoration: none;">类型</span></th>
    <th class="confluenceTh"><span style="color: rgb(51,51,51);text-decoration: none;">字段说明</span></th>
   </tr>
   <tr>
    <td class="confluenceTd">notify_event</td>
    <td colspan="1" class="confluenceTd">String</td>
    <td class="confluenceTd">通知事件</td>
   </tr>
   <tr>
    <td class="confluenceTd">notify_type</td>
    <td colspan="1" class="confluenceTd">String</td>
    <td class="confluenceTd"><p class="auto-cursor-target">通知类型</p>
     <div class="table-wrap">
      <table class="confluenceTable">
       <tbody>
        <tr>
         <th class="confluenceTh">参数</th>
         <th class="confluenceTh">说明</th>
        </tr>
        <tr>
         <td class="confluenceTd">ACQUIRE</td>
         <td class="confluenceTd">任务创建 CREATED、任务成功 SUCCESS、任务失败 FAILURE、<br />任务超时 TIMEOUT</td>
        </tr>
        <tr>
         <td class="confluenceTd">REPORT</td>
         <td class="confluenceTd">生成成功 SUCCESS、生成失败 FAILURE</td>
        </tr>
       </tbody>
      </table>
     </div></td>
   </tr>
</tbody>
</table>
</div>
        
```

这里第一个表不是规整的表，第二行少一列，只有三个td标签

可以看到有个td里还包含一个table,结构都差不多，直接用find_all()不管是table,tr还是td都会有问题

可以通过parent的parent来判断：内嵌的parent.parent应该是td标签

```
tables = b4s_body.find_all('table')
for table in tables:
    print  table 
    print   table.parent.parent.name =='td'
```

遍历tr，td,还需要校验下th,td的长度(注意查找的时候设置参数recursive=False，不查找内部的嵌套tr,td))。最后问题是如果没有th,或者th长度有问题。。。

```
b4s_body = BeautifulSoup(response.doc.html())
tables = b4s_body.find_all('table')
for table in tables:
    #print  table 
    if table.parent.parent.name =='td':
        print 'skip table: '
        print table
        continue          

    table_childrens = table.children 
    for table_children in table_childrens:
        is_tbody = table_children.name =='tbody'
        if is_tbody:
            trs = table_children.children
            for tr in trs:
                ths = tr.find_all('th',recursive=False)
                tds = tr.find_all('td',recursive=False)
                if len(ths)>0:
                    print 'len(ths)'
                    print len(ths)
                elif len(tds)>0:
                    print 'len(tds)'
                    print len(tds)
                else:
                    print 'no trs tds'
        else:
            print 'no tbody:'
            print table_children
```
最终的code

```
@config(priority=2)
def detail_page(self, response):
    # url,title,name,type,desc
    url = response.url
    title = response.doc.find('title').eq(0).text()
    #print type(response.doc)
    #print response.etree



    #print response.doc('table')
    header_array=[] 
    last_sql_data_list = []

    body = lxml.html.fromstring(response.doc('body').html())


    b4s_body = BeautifulSoup(response.doc.html())
    #print b4s_body

    #tbs = b4s_body.find_all("table", recursive=False)
    #print len(tbs)



    tables = b4s_body.find_all('table')
    last_sql_data_list = []
    for table in tables:
        #print  table 
        if table.parent.parent.name =='td':
            print 'skip table: '
            print table
            continue

        tb_thead = table.find('thead')
        th_dict ={}
        if tb_thead is not None:
            print 'tb_thead is not None'
            th_trs = tb_thead.children

            th_dict = self.get_header_dict(th_trs)
        else:
            print 'has no tag  thead: ',table
        table_childrens = table.children 
        for table_children in table_childrens:

            has_tbody = table_children.name =='tbody'
            #has_thread = table_children.name =='thead'
            if has_tbody :
                trs = table_children.children
                print 'start:',table_children.name
                print table_children

                print 'len(th_dict)',len(th_dict)

                if len(th_dict)<=0:
                    print 'find'

                    th_dict = self.get_header_dict(trs)
                #重新获取下trs
                trs = table_children.children
                td_dict_data_list = self.get_table_td_dict(trs,th_dict)
                print type(td_dict_data_list)
                sql_list_data=self.generate_sql_list_data(td_dict_data_list,url,title)
                print sql_list_data
                last_sql_data_list = last_sql_data_list + sql_list_data
            else:
                print 'no tbody:'
                print table_children
        print 'last_sql_data_list#####'
        print last_sql_data_list

    return last_sql_data_list;   


'''
1,从th中提取出
2,如果没有th，则提取class_='highlight-grey confluenceTd'
3,1,2没有，则从普通td中的第一行提取

'''
def get_header_dict(self,trs):
    th_dict = {}
    th_dict_org = {}
    for tr in trs:
        th_count = 0
        print 'get_header_dict: ',tr
        ths = tr.find_all('th',recursive=False)
        for th in ths:
            print th
        th_count =  len(list(ths))
        print th_count
        if th_count<=0:
            ths = tr.find_all('td',class_='highlight-grey confluenceTd',recursive=False)
            th_count =  len(list(ths))
            print 'find highlight-grey confluenceTd: ',th_count
        if th_count<=0:
            tds = tr.find_all('td',recursive=False)
            name = tds[0].text
            print 'name;:::::::::: ',name,len(tds)
            if name is not None and tds[0].text=='参数名':
                for i in range(len(tds)):
                    temp = tds[i].text
                    if ((temp =='参数')|(temp =='参数名')|(temp =='参数名称')|(temp =='名称')|(temp =='字段')|(temp =='字段名')|(temp=='字段名称')|(temp=='Field')):
                        if ('name' in  th_dict.values()):
                            th_dict[i]= temp
                        else:
                            th_dict[i]= 'name'
                    elif (('类型' ==temp)|(temp=='字段格式')|(temp=='参数类型')|(temp=='字段类型')|(temp=='Type')):
                        if ('type' in  th_dict.values()):
                            th_dict[i]= temp
                        else:
                            th_dict[i]= 'type'   
                    elif ((temp=='中文含义')|(temp=='含义')|(temp=='说明')|(temp=='备注')|(temp=='参数说明')|(temp=='字段说明')|(temp=='描述')|(temp=='字段描述')|(temp=='Comment')):
                        if ('desc' in th_dict.values()):
                            th_dict[i]= temp
                        else:
                            th_dict[i]= 'desc'
                    else :
                        th_dict[i]= temp
            print "from td, th_dict: "+json.dumps(th_dict, encoding="UTF-8", ensure_ascii=False)
            return th_dict



        if th_count >0:
            for i in range(th_count):
                print i
                temp = ths[i].text
                th_dict_org[i] =  temp
                print 'temp###### ',temp
                if ((temp =='参数')|(temp =='参数名')|(temp =='参数名称')|(temp =='名称')|(temp =='字段')|(temp =='字段名')|(temp=='字段名称')|(temp=='Field')):

                    if ('name' in  th_dict.values()):
                        th_dict[i]= temp
                    else:
                        th_dict[i]= 'name'
                elif (('类型' ==temp)|(temp=='字段格式')|(temp=='参数类型')|(temp=='字段类型')|(temp=='Type')):
                    if ('type' in  th_dict.values()):
                        th_dict[i]= temp
                    else:
                        th_dict[i]= 'type'   
                elif ((temp=='中文含义')|(temp=='含义')|(temp=='说明')|(temp=='备注')|(temp=='参数说明')|(temp=='字段说明')|(temp=='描述')|(temp=='字段描述')|(temp=='Comment')):
                    if ('desc' in th_dict.values()):
                        th_dict[i]= temp
                    else:
                        th_dict[i]= 'desc'
                else :
                    th_dict[i]= temp
        if th_count <=0:
            print 'no th_dict'
            print 'skip this tr',tr
        if th_count>0:
            break;
    print "th_dict: "+json.dumps(th_dict, encoding="UTF-8", ensure_ascii=False)
    print "th_dict_org: "+json.dumps(th_dict_org, encoding="UTF-8", ensure_ascii=False)
    return th_dict;

def get_table_td_dict(self,trs,th_dict):
    th_count = len(th_dict)

    td_dict_list = []
    for tr in trs:
        result = {}
        tds = tr.find_all('td',recursive=False)
        tr_children = tr.children
        td_count = len(tds)
        tr_children_count = len(list(tr_children))
        #比较下表头的长度作为校验
        if th_count <=0 or td_count !=th_count:
            print 'skip a tr',th_count,td_count
            print tr
            continue
        else:
            #print tr
            for i in range(td_count):
                td_text = tds[i].text
                result[th_dict.get(i)] =  td_text
                if td_text is None:
                    print 'tds[i]: ',tds[i]
                    result[th_dict.get(i)] = 'null'

        td_dict_list.append(result)
        print "result: "+json.dumps(result, encoding="UTF-8", ensure_ascii=False)
    print len(td_dict_list)
    return td_dict_list

def generate_sql_list_data(self,td_dict_data,url,title):
    resultlist = []
    for td_data_item in td_dict_data:
            
            templist= []
            #print len(td_data_item)
            templist.append(url)
            templist.append(title)
            fname = td_data_item.get('name')
            if fname is None :
                print "skip generate_sql_list_data: "+ json.dumps(td_data_item, encoding="UTF-8", ensure_ascii=False)
                continue
            templist.append(fname)
            ftype = td_data_item.get('type')
            if ftype is None :
                templist.append("null")
            else:
                 templist.append(ftype)
                    
            fdesc = td_data_item.get('desc')
            if fdesc is None :
                templist.append("null")
            else:
                 templist.append(fdesc)
            #print len(templist)  
            print 'llllllllllllllll'
            print '\n'.join(templist)
            resultlist.append(templist)
        #print "resultlist: "
        #print len(resultlist)
        #print  resultlist
        return resultlist
            
    def on_result(self,result):
        self.mysqlStore.insert(result)	
```
