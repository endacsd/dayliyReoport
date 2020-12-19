





### 简介



采用HttpClient实现,通过异步的方式发送http请求。

json库采用Newtonsoft。

使用方糖service,可推送打卡消息。



### 如何使用



如果需要登录，可以将用户信息添加到config.json中



```json
[
    {
        "username": "#",
        "password": "#",
        "max" : "10",
        "url": "",//用于通知
        "fields": {
            "fieldSQxq": "中心校区",//
            "fieldSQgyl": "北苑1公寓",//公寓号
            "fieldSQqsh": "1088",//寝室号
            "fieldSQnj": "2118",//学号前四位
            "fieldSQnj_Name": "2018",//入学年份
            //"fieldSQbj": "881",
            "fieldSQbj_Name": "211827"//班级名称严格按照学号前四位+班级的形式

        } 
    }
]

```







### 步骤









1. 读取用户信息
2. 尝试打卡直到停止(确定失败或打卡成功)
3. 返回对应的信息 



所需发送的表单

| url                                                  | method |
| ---------------------------------------------------- | ------ |
| https://ehall.jlu.edu.cn/sso/login                   | GET    |
| https://ehall.jlu.edu.cn/sso/login                   | POST   |
| https://ehall.jlu.edu.cn/infoplus/form/BKSMRDK/start | GET    |
| https://ehall.jlu.edu.cn/infoplus/interface/start    | POST   |
| https://ehall.jlu.edu.cn/infoplus/interface/render   | POST   |
| https://ehall.jlu.edu.cn/infoplus/interface/suggest  | POST   |
| https://ehall.jlu.edu.cn/infoplus/interface/doAction | POST   |



所需要的值

* pid
* csrfToken
* stepId
* entryId
* 回填信息
  * fieldSQbj
  * fieldSQxq
  * fieldSQgyl



#### 读取用户信息





##### 方法1



```C#
User user = new User();
user.password = "#";
user.username = "#";
user.url="#";
user.max=10;
user.dic = new Dictionary<string, string>();
user.dic.Add("fieldSQxq", "中心校区");
user.dic.Add("fieldSQgyl", "北苑1公寓");
user.dic.Add("fieldSQqsh", "1088");
user.dic.Add("fieldSQnj", "2118");
user.dic.Add("fieldSQnj_Name", "2018");
user.dic.Add("fieldSQbj","");
user.dic.Add("fieldSQbj_Name", "211827");
awit Card(user);
```



##### 方法2

```C#
var path="#";
var data=(JArray)JObject.Parse(System.IO.File.ReadAllText(path));
for(int i=0;i<data.Count;i++){
    var userInfo=data[i];
    var User=new User();
    user.username=userInfo["username"];
    user.password=userInfo["password"];
    user.max=userInfo["max"];
    user.url=userInfo["url"];
    user.dic=JsonConvert.
        DeserializeObject<Dictionary<string, Object>>(userInfo["fields"].ToString());
    //开始打打卡
    awit Card(user)
}
```



#### 尝试打卡直到结束

```c#
static async System.Threading.Tasks.Task<bool> Card(User user)
{
    //初始化HttpClient
    var handler = new HttpClientHandler() { UseCookies = true };
    var client = new HttpClient();// { BaseAddress = baseAddress };
    client.DefaultRequestHeaders.Add("Referer", "https://ehall.jlu.edu.cn/");
    bool res = false;
    for (int i = 0; i < user.max; i++)
    {
        try
        {
            bool cur = await Punch_in(client,user);
            res |= cur;
        }
        catch (Exception e)
        {
        }
        if (res == true) break;
    }
    if (res)
    {
        Console.WriteLine("打卡成功"+ user.username);
        await Commit(user, client);
    }
    else Console.WriteLine("打卡失败"+user.username);
    return res;
}
```



#### 推送

```C#
static async void Commit(User user){
    baseUrl=user.url;
    var data="text="+user.username+":ok";
    await GetAsync(baseUrl,data);
}
```



### 一些问题

1. POST表单的问题

2. 部分参数无效

3. 关于PID

   



### 打卡流程





#### 通用设置

定义需要的变量

```C#
//提取user的信息
var username = user.username;
var password = user.password;
//需要抓取的信息
var res = ""; //用于获取render表单的json
String pid = ""; //Pid
String text = "";// 每次抓取所得到的返回
String csrfToken = ""; 
String sid = "";
String entryId = "";
String form1="";
String form2="";
```





#### 登录



需要先抓取PID



```c#
//获取PID
{
    var baseUrl= "https://ehall.jlu.edu.cn/sso/login";
    text = await GetAsync(client, baseUrl);
    var reg = new Regex("(?<=name=\"pid\" value=\")[a-z0-9]{8}");
    Match _match = reg.Match(text);
    pid = _match.ToString();
    Console.WriteLine(pid);
}
```



构造好DATA然后发送



```c#
//登录
{
    var data = "username=" + username + 
        "&password=" + password;
    var baseUrl = "https://ehall.jlu.edu.cn/sso/login";
    text = await PostAsync(client, loginUrl, userInfo);
}
```



#### 获取csrfToken



```c#
//获取csrfToken
{
    var baseUrl="https://ehall.jlu.edu.cn/infoplus/form/BKSMRDK/start";
	text = await GetAsync(client,baseUrl);
    Regex reg = new Regex("(?<=csrfToken\" content=\").{32}");
    Match _match = reg.Match(text);
    csrfToken = _match.ToString();
    Console.WriteLine("csrfToken:" + csrfToken);
}
```





#### 获取流水号



```C#
//获取SID
{
    var baseUrl = "https://ehall.jlu.edu.cn/infoplus/interface/start";
    var data = "csrfToken=" + csrfToken + "&idc=BKSMRDK";
    text = await PostAsync(client, baseUrl, data);
    Regex reg = new Regex("form/\\d{8}/render");
    Match _match = reg.Match(text);
    if (!_match.Success)
    {
        return false;
    }
    sid = _match.ToString().Substring(5, 8);
    Console.WriteLine("sid:" + sid);
}
```



#### 获取信息



##### 获取data和entryId

```C#
//获取表单信息
{
    var baseUrl = "https://ehall.jlu.edu.cn/infoplus/interface/render";
    var data = "stepId=" + sid + "&csrfToken=" + csrfToken;
    res = await PostAsync(client, baseUrl, data);
    entryId = JObject.Parse(res)["entities"][0]["step"]["entryId"].ToString();
    Console.WriteLine("entryId:" + entryId);
}
```



##### 获取回填信息





```C#
//获取需要回填的信息
{
    user.bj = await getbjAsync(client, user.nj, user.bj_name, csrfToken, entryId);
    Console.WriteLine("bj:" + user.bj);
    user.dic["fieldSQbj"]= user.bj;
    user.dic["fieldSQxq"]=Getxq(user.dic["fieldSQxq"],res);
    user.dic["fieldSQgyl"]=Getgyl(user.dic["fieldSQgyl"],res);
}
```





```c#
/*
*  业务比较复杂，而且可能变化，所以单独封装了一个函数
*/
private static async System.Threading.Tasks.Task<string> getbjAsync(HttpClient client, 
                                                                    String nj, String bj,
                                                                    String csrfToken, 
                                                                    String entryId)
{
    var baseUrl = "https://ehall.jlu.edu.cn/infoplus/interface/suggest";
    var data = "prefix=&type=Code&code=BKSJKZGSB_NJ&parent=" + nj +
        "&isTopLevel=false&pageNo=0&settings=&csrfToken=" + csrfToken 
        + "&entryId=" + entryId + "&workflowId=null";
    var res = await PostAsync(client, baseUrl, data);
    var array = (JArray)JObject.Parse(res)["items"];
    for (int i = 0; i < array.Count; i++)
    {
        var codeId = array[i]["codeId"];
        var codeName = array[i]["codeName"];
        if (codeName.ToString().Equals(bj)) return codeId.ToString();
    }
    return "";
}
```



##### 获取回填信息

```c#
private static String Getxq(String xq, String data)
{
    var array = (JArray)JObject.Parse(data)["entities"][0]["codes"]["XSFX_XQ"]["items"];
    for (int i = 0; i < array.Count; i++)
    {
        var codeId = array[i]["codeId"];
        var codeName = array[i]["codeName"];
        if (codeName.ToString().Equals(xq)) return codeId.ToString();
    }
    return "";
}
private static String Getgyl(String gyl, String data)
{
    var array = (JArray)JObject.Parse(data)["entities"][0]["codes"]["BKSFXRB_GYLMC"]["items"];
    for (int i = 0; i < array.Count; i++)
    {
        var codeId = array[i]["codeId"];
        var codeName = array[i]["codeName"];
        if (codeName.ToString().Equals(gyl)) return codeId.ToString();
    }
    return "";

}


```







#### 提交打卡表单

主要是两个步骤

1. 构造表单
2. 重新编码





##### 构造表单



```C#
{
    var data = JObject.Parse(res)["entities"][0];
    var payload_1 = data["data"];
    payload_1["fieldZtw"] = "1";
    //填充payload_1
    foreach (KeyValuePair<string, string> kvp in user.dic)
    {
        payload_1[kvp.Key] = kvp.Value;
    }
    var dic = JsonConvert.DeserializeObject<Dictionary<string, Object>>(data["fields"].ToString()); 
    var payload_2 = "";
    foreach (var key in dic.Keys)
    {
        if (payload_2.Equals("")) payload_2 = key;
        else payload_2 = payload_2 + "," + key;
    }
    //得到信息
    form1 = payload_1.ToString();
    form2 = payload_2;
}
```







##### 编码并提交

```c#
//发包
{
    var data = "actionId=1&formData=" + Encode(form1) +
        "&nextUsers="+Encode("{}")+"&stepId=" + Encode(sid) +
        "&timestamp=" + 
        Encode(Convert.
               ToInt64((DateTime.UtcNow - 
                        new DateTime(1970, 1, 1, 0, 0, 0, 0)).TotalMilliseconds).
               ToString().Substring(0,10)) +
        "&boundFields=" + Encode(form2) +
        "&csrfToken=" + csrfToken;
    var baseUrl = "https://ehall.jlu.edu.cn/infoplus/interface/doAction";
    text = await PostAsync(client, baseUrl, data);      
    if (JObject.Parse(text)["ecode"].ToString().Equals("SUCCEED")) return true;
}
```





### 演示



打开成功后，会返回打开成功的流水号，可通过流水号查看信息,亦可通过推送查看打卡状态。









### 代码



```C#

using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Http;
using System.Text;
using System.Text.RegularExpressions;
namespace daily
{


    class User
    {
        public String username;   // zhengyh2118
        public String password;  // 1234567890z
        public int max;
        public String url;
        public Dictionary<String, String> dic;
    }
    class Program

    {




        static async System.Threading.Tasks.Task<bool> Punch_in(HttpClient client,User user)
        {
            //提取user的信息
            var username = user.username;
            var password = user.password;
            //需要抓取的信息
            var res = ""; //用于获取render表单的json
            String pid = ""; //Pid
            String text = "";// 每次抓取所得到的返回
            String csrfToken = "";
            String sid = "";
            String entryId = "";
            String form1 = "";
            String form2 = "";



            //获取PID
            {
                var baseUrl = "https://ehall.jlu.edu.cn/sso/login";
                text = await GetAsync(client, baseUrl);
                var reg = new Regex("(?<=name=\"pid\" value=\")[a-z0-9]{8}");
                Match _match = reg.Match(text);
                pid = _match.ToString();
                //Console.WriteLine(pid);
            }

            //登录
            {
                var data = "username=" + username + "&password=" + password;// "&pid=" + pid + "&source=";
                var baseUrl = "https://ehall.jlu.edu.cn/sso/login";
                text = await PostAsync(client, baseUrl, data);

            }


            //判断登录是否成功


            //
            //获取csrfToken
            {
                var baseUrl = "https://ehall.jlu.edu.cn/infoplus/form/BKSMRDK/start";
                text = await GetAsync(client, baseUrl);
                Regex reg = new Regex("(?<=csrfToken\" content=\").{32}");
                Match _match = reg.Match(text);
                csrfToken = _match.ToString();
                Console.WriteLine("csrfToken:" + csrfToken);
            }

            //获取SID


            {
                var baseUrl = "https://ehall.jlu.edu.cn/infoplus/interface/start";
                var data = "csrfToken=" + csrfToken + "&idc=BKSMRDK";
                text = await PostAsync(client, baseUrl, data);
                Regex reg = new Regex("form/\\d{8}/render");
                Match _match = reg.Match(text);

                if (!_match.Success)
                {
                    Console.WriteLine("无法获取StepId,可能是因为不在打卡时间内");
                    return false;
                }
                sid = _match.ToString().Substring(5, 8);
                Console.WriteLine("sid:" + sid);

            }
            //获取表单信息
            {
                var baseUrl = "https://ehall.jlu.edu.cn/infoplus/interface/render";
                var data = "stepId=" + sid + "&csrfToken=" + csrfToken;
                text = await PostAsync(client, baseUrl, data);
                res = text;
                entryId = JObject.Parse(res)["entities"][0]["step"]["entryId"].ToString();
                // Console.WriteLine("entryId:" + entryId);
            }
            //获取需要回填的信息
            {
                user.dic["fieldSQbj"]= await getbjAsync(client, user.dic["fieldSQnj"], user.dic["fieldSQbj_Name"], csrfToken, entryId);
                user.dic["fieldSQxq"] = Getxq(user.dic["fieldSQxq"], res);
                user.dic["fieldSQgyl"] = Getgyl(user.dic["fieldSQgyl"], res);
            }
            //发包
            {
                //构造表单
                {
                    var data = JObject.Parse(res)["entities"][0];
                    var payload_1 = data["data"];
                    payload_1["fieldZtw"] = "1";
                    //填充payload_1
                    foreach (KeyValuePair<string, string> kvp in user.dic)
                    {
                        payload_1[kvp.Key] = kvp.Value;
                    }
                    var dic = JsonConvert.DeserializeObject<Dictionary<string, Object>>(data["fields"].ToString()); 
                    var payload_2 = "";
                    foreach (var key in dic.Keys)
                    {
                        if (payload_2.Equals("")) payload_2 = key;
                        else payload_2 = payload_2 + "," + key;
                    }
                    //得到信息
                    form1 = payload_1.ToString();
                    form2 = payload_2;
                }
                //发包
                {
                    var data = "actionId=1&formData=" + Encode(form1) +
                        "&nextUsers="+Encode("{}")+"&stepId=" + Encode(sid) +
                        "&timestamp=" + Encode(Convert.ToInt64((DateTime.UtcNow - new DateTime(1970, 1, 1, 0, 0, 0, 0)).TotalMilliseconds).ToString().Substring(0,10)) +
                        "&boundFields=" + Encode(form2) +
                        "&csrfToken=" + csrfToken;
                    var baseUrl = "https://ehall.jlu.edu.cn/infoplus/interface/doAction";
                    text = await PostAsync(client, baseUrl, data);      
                    if (JObject.Parse(text)["ecode"].ToString().Equals("SUCCEED")) return true;
                }
            }
            return false;

        }

        static async System.Threading.Tasks.Task<bool> Card(User user)
        {

            //初始化HttpClient
            var handler = new HttpClientHandler() { UseCookies = true };
            var client = new HttpClient();// { BaseAddress = baseAddress };
            client.DefaultRequestHeaders.Add("Referer", "https://ehall.jlu.edu.cn/");

            bool res = false;
            for (int i = 0; i < user.max; i++)
            {
                try
                {
                    bool cur = await Punch_in(client,user);
                    res |= cur;
                }
                catch (Exception e)
                {
                }
                if (res == true) break;
            }


            if (res)
            {
                Console.WriteLine("打卡成功"+ user.username);
                await Commit(user, client);
            }
            else Console.WriteLine("打卡失败"+user.username);
            return res;
        }


        static async System.Threading.Tasks.Task<bool> Commit(User user,HttpClient client)
        {
            //
            //requests.get("?text="+user['username']+":jlu-daily-reporter_success")
            var baseUrl = user.url;
            var data = "text=" + user.username + ":ok";
            await PostAsync(client,baseUrl,data);
            return true;
        }

        static async System.Threading.Tasks.Task Main(string[] args)
        {




            int op = 2;
            if (op == 1)
            {
                User user = new User();
                user.password = "1234567890z";
                user.username = "zhengyh2118";
                user.url = "https://sc.ftqq.com/SCU109614T5067ef0115b2fa4e8f48506dcd1291315f38ed035c024.send";
                user.max = 1;
                user.dic = new Dictionary<string, string>();
                user.dic.Add("fieldSQxq", "中心校区");
                user.dic.Add("fieldSQgyl", "北苑1公寓");
                user.dic.Add("fieldSQqsh", "1088");
                user.dic.Add("fieldSQnj", "2118");
                user.dic.Add("fieldSQnj_Name", "2018");
                user.dic.Add("fieldSQbj", "");
                user.dic.Add("fieldSQbj_Name", "211827");
                await Card(user);
            }
            else if (op == 2)
            {
                //
                User user = new User();
                var path = @"C:\Users\EndA\source\repos\daily\user.json";
                var data = (JArray)JsonConvert.DeserializeObject(System.IO.File.ReadAllText(path));
                for (int i = 0; i < data.Count; i++)
                {

                    var userInfo = data[i];
                    var User = new User();
                    user.username = userInfo["username"].ToString();
                    user.password = userInfo["password"].ToString();
                    user.max = Convert.ToInt32(userInfo["max"].ToString());
                    user.url = userInfo["url"].ToString();
                    user.dic = JsonConvert.DeserializeObject<Dictionary<string, String>>(userInfo["fields"].ToString());
                    //开始打打卡
                    await Card(user);
                }
            }
            Console.WriteLine("quit");


        }











        private static async System.Threading.Tasks.Task<string> PostAsync(HttpClient client, String baseUrl, String data)
        {
            try
            {
                HttpResponseMessage response = await client.PostAsync(baseUrl + "?" + data, null);
                response.EnsureSuccessStatusCode();

                if (response.StatusCode != HttpStatusCode.OK) return null;
                var text = await response.Content.ReadAsStringAsync();
                return text;
            }
            catch (HttpRequestException e)
            {
                Console.WriteLine("\nException Caught!");
                Console.WriteLine("Message :{0} ", e.Message);
                return null;
            }
        }

        private static async System.Threading.Tasks.Task<string> GetAsync(HttpClient client, String baseUrl)
        {
            try
            {
                HttpResponseMessage response = await client.GetAsync(baseUrl);
                response.EnsureSuccessStatusCode();
                var text = await response.Content.ReadAsStringAsync();
                return text;
            }
            catch (HttpRequestException e)
            {
                Console.WriteLine("\nException Caught!");
                Console.WriteLine("Message :{0} ", e.Message);
                return null;
            }
        }


        private static String Encode(String str)
        {
            return System.Web.HttpUtility.UrlEncode(str.Replace("\r\n", ""), Encoding.UTF8) ;
        }
        private static String Getxq(String xq, String data)
        {
            var array = (JArray)JObject.Parse(data)["entities"][0]["codes"]["XSFX_XQ"]["items"];
            for (int i = 0; i < array.Count; i++)
            {
                var codeId = array[i]["codeId"];
                var codeName = array[i]["codeName"];
                if (codeName.ToString().Equals(xq)) return codeId.ToString();
            }
            return "";
        }
        private static String Getgyl(String gyl, String data)
        {
            var array = (JArray)JObject.Parse(data)["entities"][0]["codes"]["BKSFXRB_GYLMC"]["items"];
            for (int i = 0; i < array.Count; i++)
            {
                var codeId = array[i]["codeId"];
                var codeName = array[i]["codeName"];
                if (codeName.ToString().Equals(gyl)) return codeId.ToString();
            }
            return "";

        }


        private static async System.Threading.Tasks.Task<string> getbjAsync(HttpClient client, String nj, String bj, String csrfToken, String entryId)
        {


            var baseUrl = "https://ehall.jlu.edu.cn/infoplus/interface/suggest";
            var data = "prefix=&type=Code&code=BKSJKZGSB_NJ&parent=" + nj + "&isTopLevel=false&pageNo=0&settings=&csrfToken=" + csrfToken + "&entryId=" + entryId + "&workflowId=null";
            var res = await PostAsync(client, baseUrl, data);
            var array = (JArray)JObject.Parse(res)["items"];
            for (int i = 0; i < array.Count; i++)
            {
                var codeId = array[i]["codeId"];
                var codeName = array[i]["codeName"];
                if (codeName.ToString().Equals(bj)) return codeId.ToString();
            }
            return "";
        }
    }
}

```

