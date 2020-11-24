# modbus
## Basic
### Sensor : XY-MD02
![md](https://user-images.githubusercontent.com/42264167/99418096-3ab2de00-292d-11eb-8023-04067344b3d5.PNG)

จากภาพข้างต้น จะเห็นได้ว่าการอ่านค่าของ Temperature และ Humindity ที่อยู่ในตำแหน่ง 0x0001 และ 0x0002 ซึ่งถูกเก็บอยู่ใน input register  สามารถใช้คำสั่ง Command Register 0x04 
###Modbus serial
![](https://user-images.githubusercontent.com/42264167/99420533-e4936a00-292f-11eb-954a-1393efa89a32.PNG)

ซึ่งเราใช้ (FC=4) คือ Command Register 0x04 ที่ใช้ในการอ่าน input register ซึ่งจะรับ parameter address และ length ซึ่งระบุ address จาก 1 ไป 2 ที่มีค่า length เป็น 2 

## การเชื่อมต่อ RaspberryPi
เชื่อมต่อ Sensoe XY-MD02 เข้าสู่ Rasberry Pi ผ่าน RS485 Module ดังภาพ 
![](https://raw.githubusercontent.com/gingkasina/ModbusRTU/master/image/xy.jpeg)
เครดิตภาพ https://www.joom.com/th/products/5ca498ce28fc7101019f3549

![](https://user-images.githubusercontent.com/42264167/99426443-a8afd300-2936-11eb-8040-bbcddc504143.jpg)

## ขั้นตอนการหาว่าเชื่อมต่ออยู่ที่ไหน
สแกนหา module RS485 ที่เชื่อมต่ออยู่กับ RassberryPi
โดยใช้คำสั่ง 
``` ls /dev ``` 
โดยเชื่อมต่อ module และ ไม่ได้เชื่อมต่อ module จะมีผลลัพธ์แตกต่างกันดังนี้
![](https://raw.githubusercontent.com/gingkasina/ModbusRTU/master/image/no.PNG)
รูปนี้ ยังไม่ได้เสียบสาย 
![](https://raw.githubusercontent.com/gingkasina/ModbusRTU/master/image/have%20-%20Copy.PNG)
รูปนี้ เสียบสายแล้ว จะเห็นถึงความแตกต่างที่ วงกลมสีแดง คือมี ttyUSB0 เพิ่มขึ้นมา 

## express ทำการตั้งค่า web request
ในขั้นตอน express ทำการตั้งค่า web request ดังนี้
``` js
var express=require('express');
var morgan=require('morgan');
var compression=require('compression');
var bodyParser=require('body-parser');

module.exports=()=>{
    var app=express();
    app.use(morgan('dev'));
    app.use(bodyParser.urlencoded({
	extended: true
    }));
    app.use(express.static('./public'));
    app.use(bodyParser.json());
    app.set('views','./app/views');
    app.set('view engine', 'jade');
    require('../app/routes/index.route')(app);
    require('../app/routes/api.route')(app);
    return app;
}
```
## ติดตั้ง modbus serial
ทำการติดตั้ง modbus-serial ผ่านคำสั่ง 
```npm install modbus-serial --save  ```
แล้วทำการเชื่อมต่อไปยัง sensor โดยใช้คำสั่ง ดังต่อไปนี้
``` js
var ModbusRTU=require('modbus-serial');
var client=new ModbusRTU();
client.connectRTU("/dev/ttyUSB0",{buadRate: 9600});
```
โดย /dev/ttyUSB0 คือพอร์ต ของ RS485 

จากนั้นทดลองทำการอ่านค่า โดยใช้คำสั่ง 
``` js
client.readInputRegisters(1,2).then((data)=>{
    console.log(data);
}).catch((e)=>{
    console.log(e.message);
});
```

เมื่อทำการอ่านค่าได้แล้ว จึงต้องเอาค่าไปเก็บไว้ที่ดาต้าเบส 
แล้วจึงทำการประกาศค่า modbusRTU เป็น readInputRegisters
และ ให้ทำการอัพเดตทุกๆ 60000 m/s หรือ 1 นาที
``` js
setInterval(()=>{
    modbusRTU.readInputRegisters(1,2)
    .then((data)=>{
        var allData=data.data;
        var model={
            temperature: allData[0]/10,
            humidity: allData[1]/10,
            datetime: Date.now()
        }
        modbusModel.insertMany(model,(err,docs)=>{
            if(err)console.log(err);
            // else console.log(docs)
        });
    }).catch((e)=>{
        console.log(e.message);
    })
},60000); 
``` js

ตั้งค่าหน้าเว็บ 
ในส่วนของ models จะเก็บค่า อุณหภูมิ ความชื้น และวันเวลา
```
var modbusSchema=new Schema({
    temperature: Number,
    humidity: Number,
    datetime: String
});
```

api.controller เป็นฟังก์ชั่นการทำงานที่จะรีเทิร์นค่าอุณหภูมิย้อนหลัง
``` js
var modbus=(req,res)=>{
    modbusModel.find({},(err,data)=>{
        if(!err){
            let detail=new Array();
            let tmp={};
	        data.sort((a,b)=>{
		        return b.datetime-a.datetime;
	        });
            data.forEach((item,index)=>{
		        if(!item.temperature||!item.humidity||item.humidity>100||item.temperature>100)return;
                let dt=new Date(parseInt(item.datetime));
                detail.push({'dateTime':dt, 'temperature':item.temperature, 'humidity':item.humidity});
            });
            detail.forEach((item,index)=>{
                if(isNaN(item.dateTime.getUTCDate())||isNaN(item.dateTime.getUTCMonth())||isNaN(item.dateTime.getUTCFullYear())||!item.temperature||!item.humidity)return;
		        let obj=tmp[item.dateTime.getUTCDate()+'/'+(item.dateTime.getUTCMonth()+1)+'/'+item.dateTime.getUTCFullYear()]=tmp[item.dateTime.getUTCDate()+'/'+(item.dateTime.getUTCMonth()+1)+'/'+item.dateTime.getUTCFullYear()]||{count:0, totalTemperature:0, totalHumidity:0};
		        obj.count++;
		        obj.totalTemperature+=item.temperature;
		        obj.totalHumidity+=(item.humidity/5);
            });
	        let result=Object.entries(tmp).map(function(entry){
		        return {date: entry[0], temperature: entry[1].totalTemperature/entry[1].count, humidity: entry[1].totalHumidity/entry[1].count};
	        });
            res.json({detail:detail, average: result.slice(0,7)});
        }
    })
}
```
ในหน้า index ให้ดึงดาต้าเบสขึ้นมา
``` js
var render=(req,res)=>{
    modbusModel.find({},(err,data)=>{
	    if(!err)res.render('index',{tempData: data});
    });
}
```
``` js
var render=(req,res)=>{
    modbusModel.find({},(err,data)=>{
	    if(!err)res.render('index',{tempData: data});
    });
}
```

เป็นการทำงานของ api ที่เมื่อ /api ให้ขึ้น rander แต่เมื่อ /api/modbus ให้ขึ้น api.modbus
``` js
module.exports=(app)=>{
    var api=require('../controllers/api.controller');
    app.get('/api',api.render).get('/api/modbus',api.modbus);
}
```
``` js
module.exports=(app)=>{
    var index=require('../controllers/index.controller');
    app.get('/',index.render);
}
```
จากนั้นก็ดึงข้อมูลดาต้าเบส ออกมา ด้วย mongoose

``` js
var mongoose=require('mongoose');

module.exports=()=>{
    mongoose.set('debug',true);
    var db=mongoose.connect('mongodb://localhost/modbus');
    require('../app/models/modbus.model');
    return db;
}
``` 
สร้างหน้าเว็บขึ้นมา เพื่อแสดงผลลัพธ์ดาต้าเยส ของ อุณหภูมิ ความชื้น รวมถึงวันและเวลา
``` 
doctype html
html(lang="en")
    head
        meta(charset='utf-8')
        meta(name='viewport', content='width=device-width, initial-scale=1.0')
        link(rel="stylesheet", href="/lib/angular-datatables/dist/css/angular-datatables.min.css")
        link(rel="stylesheet", href="/lib/datatables.net-dt/css/jquery.dataTables.min.css")
    body(ng-app='app')
        h1(style='text-align: center; color: blue;') Temperature and Humidity Monitoring.
        div(ng-controller='LineCtrl')
            canvas(class="chart chart-line" chart-data="data" chart-labels="labels" chart-series="series" chart-options="options" chart-click="onClick")
        div(ng-controller='DatatablesCtrl')
            //- p {{ details }}
            table(datatable="ng" dt-options="dtOptions" dt-column-defs="dtColumnDefs" dt-instance="dtInstance" class="table table-striped table-bordered")
                thead
                    tr
                        th #
                        th Temperature
                        th Humidity
                        th Datetime
                tbody
                    tr(ng-repeat="detail in details")
                        th {{ $index+1 }}
                        th {{ detail.temperature }}
                        th {{ detail.humidity }}
                        th {{ detail.dateTime | formatAsDate }}
    script(src="/lib/jquery/dist/jquery.min.js")
    script(src="/lib/datatables.net/js/jquery.dataTables.min.js")
    script(src="/lib/chart.js/dist/Chart.min.js")
    script(type='text/javascript', src='/lib/angular/angular.min.js')
    script(src="/lib/ang
    ular-datatables/dist/angular-datatables.min.js")
    script(src="/lib/angular-chart.js/angular-chart.js")
    script(src="/lib/moment/min/moment.min.js")
    script(type='text/javascript', src='/tempChart.js')
    ```

