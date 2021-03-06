## 前言

因为疫情影响[机械工业出版社在线阅读免费](http://ebooks.cmanuf.com/)，恰好其又有大量的计算机的好书，所以就心生了把计算机方向的所有的书都下载下来的念头。（至于整个网站的书，硬盘存不下啊2333（现在这个网站接口开着，但是web访问不了，所以凭印象写。

按照普通用户单次转存500的上限 分了三个文件夹
链接：https://pan.baidu.com/s/1ZeND0YxnTk6Gc2RQFWsh-Q     提取码：xhcw 

## 观察

随便点开几本书，有的图书提供了pdf阅读，有的书提供了在线web阅读。估计是因为有的书没有文字资料所以才提供了pdf阅读。

### 下载PDF阅读的图书

对于pdf下载，稍微观察下载链接可以知道，其下载链接的url是由ISBN+版次号决定的，感谢同学cyr把所有的图书的isbn和版次号都爬下来了，具体方法观察网站的接口大概可以知道，盲猜不难（（

isbn和版次号爬下来以后去重，根据链接格式就可以生成一份链接合集，用idm批量下载。文件为[pdf.txt ](https://github.com/yqylh/-Reptile/blob/master/pdf.txt)。下载完以后还有一个问题，该网站命名文件是按照isbn+版次号命名的，虽然有利于他们的管理，但是不例如我们查找图书，于是该同学给了我另外一份文件[pdf2.txt](https://github.com/yqylh/-Reptile/blob/master/pdf2.txt) 其内为 推算出的文件名|书名，于是我写了一个自动rename的[C++程序](https://github.com/yqylh/-Reptile/blob/master/rename.cpp)

```c++
    std::string s;
    while(std::getline(std::cin, s)) {
        std::cout << s.substr(0, s.find_first_of('|')) << std::endl;
        std::cout << s.substr(s.find_first_of('|') + 1, s.length() - s.find_first_of('|') - 1) << "\n";
        
        std::string oldName = "./pdf/" + s.substr(0, s.find_first_of('|')) + "_2.pdf";
        std::string newName = "./pdf/" + s.substr(s.find_first_of('|') + 1, s.length() - s.find_first_of('|') - 1) + ".pdf"; 
        if (!rename(oldName.c_str(), newName.c_str())) {
            std::cout << "rename success "<< std::endl;
        } else {
            std::cout << "rename error "<< std::endl;
        }
    }
```

自此 所有pdf格式的书都已经下载完毕了，共611本31.9GB。

### 爬取H5阅读的图书

计算机方面的书一共有1400本左右，其中pdf有611本，那大约有800本只提供H5阅读的图书，之所以断定只提供H5，是因为这些图书根据isbn和版次号下载的话会提示无权限下载。那么就想办法把这本书爬取下来，观察了很久发现没有任何接口可以提供我们直接下载这本书的文本资源。

首先点开一本书的在线阅读页面，从左侧的目录开始点章节会发现他会给一个叫readbook的接口发送请求，这个接口会把一个XHTML发送回来，而其格式为

`www.hzcourse.com/resource/readBook?path=/openresources/teach_ebook/uncompressed/+id+/OEBPS/Text/chapter?.xhtml `

这里面有两个关键点，一个是id，也就是一串数字，另一个是chapter?.xhtml。随缘在网页的代码搜索了一下，找到了一个变量名为'ebookid'，其的值就是这个id，另一个chapter应该是章节的编号。

那接下来的逻辑就很明显了

1. 找到所有H5阅读的书的id
2. 找到每本书的list，即chapter？
3. 根据每本书list下载所有的XHTML连接成一个HTML输出
4. 修正，导出pdf

#### 找到所有H5阅读的书的id

（开始是由qer找所有的H5书的id，但是不知道他的方法是啥，才找了一点就被限制访问了，继续鞭尸qer，由于我采用的方法直接调用的是阅读，不需要任何验证，所以不会被限制访问，而且qer给我的还是有重复的，我又写了去重，这在[downH5.cpp](https://github.com/yqylh/-Reptile/blob/master/downH5.cpp)里 懒得再说了2333）

多打开几本书，查看接口调用可以发现，每本书一定有一个文件，其为`'http://www.hzcourse.com/resource/readBook?path=/openresources/teach_ebook/uncompressed/'+temp+'/OEBPS/Text/chapter.xhtml'` 然后每本书的id基本都是五位数，从网站的存书量看id不会超过100000，于是开始从0到100000开始枚举是否存在这个文件。程序[getCanReadList](getCanReadList.js)如下

```js
var request = require('request');
download = (temp)=> {
    request('http://www.hzcourse.com/resource/readBook?path=/openresources/teach_ebook/uncompressed/'+temp+'/OEBPS/Text/chapter.xhtml', (err, data)=>{
        if (err) {
            console.log(err);
            return;
        }
        if (data.body != "Failed to load resource: File Not Found") {
            console.log(temp)
        }
    })
}
let i = 0;
setInterval(()=>{
    console.log(i);
    download(++i);
}, 75)
```

如果时间间隔太短，请求会炸裂，程序就跪了所以设置了75ms的时间间隔。

#### 找到每本书的list，即chapter？

我忘记我是啥时候看到的queryAllChapterList这个接口了，应该是看网络请求的时候发现的。其参数为ebookid和token，从其对应的就是这个书的id和阅读者的一个token（约1h会失效），同时还有cookie验证。我开始尝试用node发送这个请求，但我始终找不到办法获取token，于是我转变思路，从前端发送请求，把接口按照一定的格式输出，在后端再进行处理。

按F12在代码里能找到这个接口的调用代码，格式大约是(自己魔改以后的)：

```js
ebookId = $.trim($("#ebookId").val());
token = $.trim($("#token").val());
str = ""
jQuery.ajax({
    type : "post" , 
    url : "web/refbook/queryAllChapterList", 
    dataType : "json" , 
    data : {ebookId:ebookId,token:token},
    success: (test)=>{
        test = test.data.data
        console.log(test[0].ebookName + " " + ebookId)
        str += '['
        for (i of test) {
            str += '"http://www.hzcourse.com/resource/readBook?path=' + i.ref + '",'
            var url = "resource/readBook?path="+i.ref;
            jQuery.ajax({
                type : "get" , 
                url : url,
                success: (data)=>{
                    console.log(data)
                }
            })
        }
        str += ']'
        console.log(str)
    }
});
```

这个接口返回了这本书每个章节的链接，这个时候在调用readBook接口就能把每个的data链接起来，重命名一个html，成功获得这本书的全部文本内容了，之后要考虑自动化的方式了（笑，其实噩梦才刚刚开始。

#### 根据每本书list下载所有的XHTML连接成一个HTML输出

由于XHTML文本内容很多，所以根据我暴力跑出来的所有H5图书的id，对每本书调用queryAllChapterList把list输出出来，按照`[[name][list[url, url]],[name][list[url, url]]]`的三维数组的方式输出，这样我只要把输出结果复制过去，稍加修改，就变成了三维数组的定义复制代码了。

这时遇到一个问题，有的书在调用接口的时候被拒绝访问，开始以为可能是暴力写错了，但是我确实可以下载那个xhtml，怀疑是有的书没有开放阅读，无法获取list，但确实存在这本书的文本内容，于是我写了另一个脚本来判断其是可以获取list,如果可以获取list就按照上文说的三维数组定义的格式输出。

```js
mmdm = ['1136','1481','1488','1497','1500','1505','1507','1508','1515','1524','1525','1526','1527','1534','1538','1544','1545','1546','1547','1552','1550','1553','1555','1558','1559','1561','1564','1567','1568','1571','1572','1574','1629','1630','1632','1641','1646','1647','1648','1650','1654','1655','1693','1694','1695','1703','1704','1705','1708','13075','13278','13441','13743','14182','14325','14395','14505','14672','14783','14850','14875','14876','14877','14878','14881','14883','14884','14885','14886','14889','14898','14899','14900','14902','14903','14915','14914','14917','14920','14923','14934','14936','14939','14941','14943','14947','14950','14957','14958','14962','14964','14968','14971','14974','14977','14978','14988','14992','14998','14999','15005','15008','15013','15014','15015','15016','15020','15022','15023','15029','15030','15031','15033','15035','15034','15036','15037','15038','15040','15045','15047','15049','15050','15052','15054','15059','15064','15066','15070','15071','15072','15079','15082','15087','15088','15089','15092','15101','15105','15106','15107','15108','15114','15119','15121','15122','15125','15128','15132','15134','15137','15139','15140','15146','15148','15149','15153','15154','15155','15158','15161','15162','15166','15170','15174','15179','15185','15186','15188','15187','15195','15202','15207','15214','15216','15220','15257','15284','15287','15288','15290','15291','15292','15294','15295','15298','15304','15305','15306','15307','15309','15308','15310','15319','15320','15323','15324','15325','15326','15327','15328','15329','15331','15332','15336','15337','15338','15339','15341','15342','15344','15345','15346','15347','15352','15353','15358','15361','15364','15366','15369','15372','15371','15373','15376','15378','15377','15380','15384','15392','15391','15393','15394','15395','15398','15401','15403','15404','15405','15410','15413','15412','15417','15418','15421','15423','15425','15426','15427','15430','15431','15436','15439','15440','15441','15442','15444','15447','15449','15452','15453','15454','15455','15457','15456','15461','15464','15467','15468','15469','15472','15471','15489','15487','15492','15493','15495','15496','15497','15498','15501','15509','15510','15513','15516','15515','15519','15520','15524','15525','15528','15532','15534','15535','15536','15538','15541','15544','15547','15549','15554','15555','15558','15560','15570','15572','15573','15574','15575','15576','15577','15578','15579','15580','15581','15583','15586','15590','15597','15601','15603','15605','15610','15611','15615','15620','15621','15627','15629','15631','15633','15635','15636','15638','15639','15642','15644','15645','15649','15652','15653','15654','15657','15658','15663','15664','15665','15667','15669','15670','15671','15673','15678','15679','15681','15677','15676','15685','15686','15689','15699','15698','15700','15702','15701','15704','15708','15714','15717','15718','15723','15725','15728','15729','15735','15738','15740','15742','15743','15744','15746','15750','15752','15753','15755','15756','15759','15764','15765','15766','15768','15769','15770','15777','15776','15779','15781','15784','15787','15788','15794','15793','15795','15796','15803','15807','15806','15808','15818','15820','15823','15824','15826','15828','15829','15830','15832','15833','15838','15840','15845','15847','15850','15852','15857','15858','15859','15864','15866','15868','15871','15873','15876','15877','15878','15881','15882','15887','15891','15893','15897','15917','15919','15920','15921','15922','15923','15924','15925','15928','15929','15931','15932','15933','15934','15935','15936','15938','15940','15942','15943','15946','15948','15950','15951','15952','15953','15954','15956','15960','15966','15969','15977','15980','15978','15981','15982','15983','15994','15997','16000','16001','16002','16003','16007','16008','16010','16011','16012','16014','16017','16018','16020','16021','16022','16023','16024','16025','16027','15831','16033','16034','16036','16037','16038','16039','16041','16042','16045','16046','16047','16049','16057','16058','16060','16061','16063','16064','16067','16068','16073','16074','16075','16076','16077','16080','16079','16081','16082','16083','16085','16084','16086','16087','16088','16090','16092','16095','16099','16100','16101','16104','16108','16109','16110','16114','16118','16117','16119','16122','16123','16125','16136','16179','16180','16181','16183','16286','16289','16288','16291','16295','16297','16300','16302','16303','16305','16308','16309','16317','16318','16319','16320','16323','16326','16328','16329','16331','16334','16335','16337','16338','16340','16341','16344','16346','16345','16347','16350','16351','16356','16355','16357','16358','16360','16362','16363','16375','16377','16378','16379','16381','16382','16383','16384','16385','16388','16395','16396','16401','16404','16412','16411','16413','16418','16420','16421','16423','16424','16425','16429','16430','16433','16437','16440','16443','16444','16445','16446','16447','16453','16456','16467','16472','16477','16479','16478','16480','16481','16482','16488','16490','16508','16515','16524','16525','16528','16534','16535','16537','16540','16542','16545','16547','16550','16555','16569','16570','16571','16572','16575','16576','16578','16579','16463','16464','17156','17158','17163','17169','17170','17173','17174','17176','17218','17219','17222','17225','17229','17228','17230','17232','17237','17238','17239','17241','17243','17247','17251','17252','17257','17260','17261','17265','17266','17267','17271','17272','17274','17273','17280','17282','17283','17296','17297','17302','17304','17307','17331','17332','17334','17336','17338','17339','17346','17347','17352','17354','17376','17380','17381','17396','17395','17400','17405','17413','17414','17422','17429','17432','17433','17436','17437','17438','17439','17445','17447','17449','17450','17452','17459','17460','17461','17466','17467','17469','17473','17479','17480','17481','17485','17489','17497','17522','17529','17542','17549','17555','17561','17567','17576','17577','17581','17588','17589','17593','17596','17597','17609','17612','17613','17615','17619','17632','17642','17644','17652','17656','17665','17678','17679','17683','17686','17688','17691','17697','17708','17709','17710','17711','17720','17722','17729','17736','17737','17738','17746','17758','17768','17771','17775','17776','17782','17790','17795','17817','17819','17828','17829','17833','17831','17842','17844','17872','17879','17882','17883','17888','17893','17898','17905','17930','17933','17946','17950','17952','17955','17958','17966','17967','17971','17972','17973','17979','17981','17983','17986','17991','17990','17994','17996','18006','18010','18012','18013','18015','18014','18029','18041','18042','18045','18053','18052','18054','18056','18058','18062','18104','18271','18285','18356','18359','18381','18448','18464','18508','18512','18531','18532','18561','18564','18579','18588','18549']
token = $.trim($("#token").val());
out = ""
download = (id) =>{
    return new Promise((resolve, reject) =>{
        ebookId = id;
        str = ""
        jQuery.ajax({
            type : "post" , 
            url : "web/refbook/queryAllChapterList", 
            dataType : "json" , 
            data : {ebookId:ebookId,token:token},
            success: (test)=>{
                if (test.data.code == 2) {
                    resolve()
                    return;
                }
                test = test.data.data
                str += '["' + test[0].ebookName + " " + ebookId + '",'
                str += '['
                for (i of test) {
                    str += '"' + i.ref + '",'
                }
                str += ']],'
                out += str
                resolve()
            }
        });
    })
}
main = async (a, b) =>{
    for (let i = a; i < b; i++) {
        console.log(i)
        await download(mmdm[i]);
    }
}
main(0,mmdm.length)
```

以上两段代码均在[getCanDownList](https://github.com/yqylh/-Reptile/blob/master/getCanDownList.js)里。最开始获取的所有的id（3000个左右），筛去不能看的，剩下了800多个，估算了一下恰好+600是一千四左右，恰好是计算机图书的数量，难道只有计算机的书有H5阅读的方式？我怀疑网站在搞我。

在有了这800+个id对应的所有下载链接以后，开始写脚本下载输出html，这里面有好多坑qnq，先把[最后的代码](https://github.com/yqylh/-Reptile/blob/master/downH5.js)放上去

```js
var request = require('request');
var fs = require('fs');

download = (temp)=> {
    return new Promise((resolve, reject) => {
        try{
            let name = temp[0].substr(0, temp[0].length - 6);
            name = name.replace(/\//g, '-');
            name = name.replace(/:/g, '-');
            let id = temp[0].substr(temp[0].length - 5, 5)
            // console.log(name, id)
            filearr = temp[1]
            let book = new Array(filearr.length)
            num = 0
            for (let i = 0; i < filearr.length; i++) {
                function getBook(k) {
                    request('http://www.hzcourse.com/resource/readBook?path='+filearr[i], (err, data)=>{
                        book[k] = data.body
                        num++
                        if (num == filearr.length) {
                            let result = ""
                            book.forEach(i=>{result += i})
                            var result1 = result.replace(/\.\./g, 'http://www.hzcourse.com/resource/readBook?path=/openresources/teach_ebook/uncompressed/' + id +'/OEBPS/Text/..');
                            var c = 'http://www.hzcourse.com/resource/readBook?path=/openresources/teach_ebook/uncompressed/' + id +'/OEBPS/Text/../Styles/main.css'
                            result = result1.replace(c, '../Styles/main.css')
                            result = result.replace(/uncompressed\/\s/g, 'uncompressed/')
                            console.log(name, id)
                            fs.writeFile(__dirname + '\\html\\' + name + '.html', result, (err)=>{
                                if (err) console.log(err);
                                resolve();
                            })
                        }
                    })
                }
                getBook(i)
            }
        } catch(err) {
            console.log(err);
            resolve()
        }
    })
}
main = async () =>{
    // 多线程
    // let donum = process.argv.splice(2)[0]
    // download(a[donum])
    // console.log(donum)
    // 针对单个
    // for (let i = 0; i < a.length; i++) {
    //     if (a[i][0].indexOf('10倍速目标达成法') != -1) {
    //         await download(a[i])
    //         console.log(i)
    //     }
    // }
    // 补全
    // for (let i = 871; i < 874; i = i + 1) {
    //     let ans = await new Promise((resolve)=>{
    //         let name = a[i][0].substr(0, a[i][0].length - 5);
    //         let exist = fs.existsSync(__dirname + '\\html\\' + name + '.html')
    //         if (exist) {
    //             resolve(1)
    //         }
    //         name = name.replace(/\//g, '-');
    //         exist = fs.existsSync(__dirname + '\\html\\' + name + '.html')
    //         if (exist) {
    //             resolve(1)
    //         }
    //         name = name.replace(/:/g, '-');
    //         exist = fs.existsSync(__dirname + '\\html\\' + name + '.html')
    //         if (exist) {
    //             resolve(1)
    //         }
    //         resolve(0)
    //     })
    //     if (ans == 1) {
    //         console.log(i+"号存在")
    //     } else {
    //         console.log(i+"号不存在，正在下载")
    //         await download(a[i])
    //         console.log("下载完毕")
    //     }
    // }
    // 全部下载
    // i = 795
    // setInterval(()=>{
    //     console.log(i);
    //     download(a[i++]);
    // }, 10000)
    // 修复
    // str = "["
    // fs.readdir(__dirname + '/html2', (err, file) =>{
    //     file.forEach(files =>{
    //         for(let i = 0; i < a.length; i++) {
    //             let name = a[i][0].substr(0, a[i][0].length - 5);
    //             name = name.replace(/\//g, '-');
    //             name = name.replace(/:/g, '-');
    //             if (files.indexOf(name) != -1) {
    //                 str += i + ',';
    //                 console.log(str)
    //                 break;
    //             }
    //         }
    //     })
    // })
    // c = [496,484,747,733,390,385,315,734,722,675,816,307,727,545,409,456,305,770,644,475,118,501,407,864,786,715,417,724,673,682,847,762,656,657,758,405,854,35,548,670,423,702,794,737,503,723,795,629,728,785,744,391,712,194,463,826,678,396,274,168,606,300,823,317,718,739,835,661,667,645,840,650,710,380,568,688,551,418,602,594,788,745,658,743,649,764,584,517,481,604,323,469,476,490,590,776,852,433,663,399,735,319,570,483,485,680,245,627,720,841,642,691,515,290,586,309,301,512,779,383,304]
    // for (let i = 113; i < c.length; i++) {
    //     await download(a[c[i]])
    // }
}
main()
```

其中还有a的定义，就是那个三维数组，实在是太长了，就不放了。看main的复杂程度可以看出我真的写了好多种方式……因为这个网站他太不稳定了，一丢包程序就卡住或者崩溃了，所以超级烦。下面是各种坑：

1. 有的书他的名字含有/:这种特殊字符，这些在Windows里都是不能出现的，所以我的程序爆炸了好多次才照清楚原因，解决方法：
```js
let name = temp[0].substr(0, temp[0].length - 6);
name = name.replace(/\//g, '-');
name = name.replace(/:/g, '-');
```
在从三维数组获取名字以后，把所有的/:替换成-
至于空格，（开始的时候我算错了名字的长度，所以所有的名字都是以空格结尾）为了方便后序的处理，我写了脚本进行rename,[mkpdf.js](https://github.com/yqylh/-Reptile/blob/master/mkpdf.js)

```js
fs.readdir(__dirname + '/temp', async (err, file) =>{
    console.log(file.length)
    file.forEach(i =>{
        // i = i.replace(/\s/g, '');
        fs.rename(__dirname + '/temp/' + i, __dirname + '/temp/' + i.replace(/\s/g, ''), err => {
            if (err) {
                console.log(err);
                return
            }
            console.log('update success')
        })
    })
})
```

2. 

```js
var fs = require('fs')
    str = "["
fs.readdir(__dirname + '/html', async (err, file) =>{
    console.log(file.length)
    file.forEach(file =>{
        let url = __dirname + '/html/' + file
        fs.readFile(url, (err, data) =>{
            if (err) {console.log(err); return;}
            if (data.indexOf('The proxy server received an invalid')!= -1) {
                for(let i = 0; i < a.length; i++) {
                    let name = a[i][0].substr(0, a[i][0].length - 6);
                    name = name.replace(/\//g, '-');
                    name = name.replace(/:/g, '-');
                    if (file.indexOf(name) != -1) {
                        str += i + ',';
                        console.log(str)
                        break;
                    }
                }
            }
        })
    })
})
```

这段[脚本](https://github.com/yqylh/-Reptile/blob/master/downH5.js)是因为有的时候因为网络问题，有些章节会下载失败，我遍历了所有的文件，对于含有*The proxy server received an invalid*的要重新下载。

3. 在连接时，因为不同章节的请求完成的时间不一定相同，request又是异步函数，于是写了getBook调用request函数，传参数k，在request完成后，把结果放到数组第K为，然后当所有的结果都获取了以后，按照数组顺序连起来。这时还有一个问题，每个XHTML里面的图片都用的网站的相对位置，需要根据请求的接口全都替换成网络绝对位置（按照我写的代码会把css的位置也替换掉，但是所有的css都是一样的，我就提前下好了一份放在了Styles文件夹了，所以还需要再把css换成相对位置）特别的是，有的XHTML会出现问题 例如`uncompressed \`中间多了一个空格，所以要全都替换掉。具体实现如下

```js
let result = ""
book.forEach(i=>{result += i})
var result1 = result.replace(/\.\./g, 'http://www.hzcourse.com/resource/readBook?path=/openresources/teach_ebook/uncompressed/' + id +'/OEBPS/Text/..');
var c = 'http://www.hzcourse.com/resource/readBook?path=/openresources/teach_ebook/uncompressed/' + id +'/OEBPS/Text/../Styles/main.css'
result = result1.replace(c, '../Styles/main.css')
result = result.replace(/uncompressed\/\s/g, 'uncompressed/')
```

4. 最终按照名字写入到html里，存在本地

#### 修正，导出pdf

因为有的时候网络问题，会导致一些文件最后没有输出，所以还需要查找所有缺了的文件重新下载

```js
for (let i = 1; i < 874; i = i + 1) {
    let ans = await new Promise((resolve)=>{
        let name = a[i][0].substr(0, a[i][0].length - 5);
        let exist = fs.existsSync(__dirname + '\\html\\' + name + '.html')
        if (exist) {
            resolve(1)
        }
        name = name.replace(/\//g, '-');
        exist = fs.existsSync(__dirname + '\\html\\' + name + '.html')
        if (exist) {
            resolve(1)
        }
        name = name.replace(/:/g, '-');
        exist = fs.existsSync(__dirname + '\\html\\' + name + '.html')
        if (exist) {
            resolve(1)
        }
        resolve(0)
    })
    if (ans == 1) {
        console.log(i+"号存在")
    } else {
        console.log(i+"号不存在，正在下载")
        await download(a[i])
        console.log("下载完毕")
    }
}
```

下载XHTML的大坑基本就以上，其他的没说到的代码基本是因为发现了bug，所以进行修正的代码233，就不在陈述了。

对于输出pdf，本来是用的AdobeAcrobat进行批量导出，但是这个垃圾软件只有32位版本，而且它继承了psprae的特性，越用占用内存越高，所以输出十个左右就自动崩溃了，没有任何解决办法。由于有八百多个HTML，无法用在线工具解决，特别是里面的图片都是按照图片链接存储的，万一哪天这个网站更新了接口就gg，在试了很多种方法以后 最后选定了wkhtmltopdf，这是一个命令行工具，可以转换pdf，非常好用！最后的脚本[mkpdf.js](mkpdf.js)，控制输出所有的pdf。

```js
var fs = require('fs')
var exec = require('child_process').exec;

fs.readdir(__dirname + '/html', async (err, file) =>{
    console.log(file.length)
    for (var i = 0; i < file.length; i++) {
        console.log('wkhtmltopdf.exe ' + __dirname + '/html/'+ file[i] + ' ' + __dirname + '/html/'+ file[i] + '.pdf')
        todo = () =>{
            return new Promise(resolve =>{
                exec('wkhtmltopdf.exe ' + __dirname + '/html/'+ file[i] + ' ' + __dirname + '/html/'+ file[i] + '.pdf', (error, stdout, stderr) => {
                    console.log(error, stdout, stderr)
                    resolve()
                })
            })
        }
        await todo();
    }
})
```