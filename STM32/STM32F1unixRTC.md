# 1. STM32F1系列单片机_UNIX时间戳
## 1. 什么是Unix时间戳
    Unix时间戳是从1970年1月1日（UTC/GMT的午夜）开始所经过的秒数，不考虑闰秒。
    Unix时间戳的0按照ISO 8601规范为 ：1970-01-01T00:00:00Z.
    一个小时表示为UNIX时间戳格式为：3600秒；一天表示为UNIX时间戳为86400秒，闰秒不计算。
    在大多数的UNIX系统中UNIX时间戳存储为32位，这样会引发2038年问题或Y2038
    通俗的讲， 时间戳是一份能够表示一份数据在一个特定时间点已经存在的完整的可验证的数据。
    （简单来讲：就是从1970年1月1日开始到你看到这片笔记时经过的秒数）
## 2. STM32F1 如何实现Unix 时间戳
*直接上程序 程序是基于Hal库来实现的*
-  2.1 UnixRtc.c
    ``` c
    #include "unixrtc.h"
    _calendar_obj calendar;//时钟结构体 
    //月份数据表											 
    const uint8_t table_week[12]={0,3,3,6,1,4,6,2,5,0,3,5}; //月修正数据表	 
    //平年的月份日期表
    const uint8_t mon_table[12]={31,28,31,30,31,30,31,31,30,31,30,31};


    void rtc_init_user(void)
    {	
        HAL_RTCEx_SetSecond_IT(&hrtc);//秒中断使能，没有配置这个中断可以不加
        if(HAL_RTCEx_BKUPRead(&hrtc,RTC_BKP_DR1)!=0x5050)//是否第一次配置
        {
            RTC_Set(2019,4,7,16,28,0); //设置日期和时间							  
            HAL_RTCEx_BKUPWrite(&hrtc,RTC_BKP_DR1,0x5050);//标记已经初始化过了
            printf("---first Time---\n");
        }
        RTC_Get();//更新时间
    }
    
    void RTC_Set(uint16_t syear,uint8_t smon,uint8_t sday,uint8_t hour,uint8_t min,uint8_t sec)
    {
        uint16_t t;
        uint32_t seccount=0;
        
        if(syear<1970||syear>2099)	return; 
        for(t=1970;t<syear;t++)	//把所有年份的秒钟相加
        {
            if(Is_Leap_Year(t))seccount+=31622400;//闰年的秒钟数
            else seccount+=31536000;			  //平年的秒钟数
        }
        smon-=1;
        for(t=0;t<smon;t++)	   //把前面月份的秒钟数相加
        {
            seccount+=(uint32_t)mon_table[t]*86400;//月份秒钟数相加
            if(Is_Leap_Year(syear)&&t==1)seccount+=86400;//闰年2月份增加一天的秒钟数	   
        }
        seccount+=(uint32_t)(sday-1)*86400;//把前面日期的秒钟数相加 
        seccount+=(uint32_t)hour*3600;//小时秒钟数
        seccount+=(uint32_t)min*60;	 //分钟秒钟数
        seccount+=sec;//最后的秒钟加上去	
    
        //设置时钟
        RCC->APB1ENR|=1<<28;//使能电源时钟
        RCC->APB1ENR|=1<<27;//使能备份时钟
        PWR->CR|=1<<8;    //取消备份区写保护
        //上面三步是必须的!
        RTC->CRL|=1<<4;   //允许配置 
        RTC->CNTL=seccount&0xffff;
        RTC->CNTH=seccount>>16;
        RTC->CRL&=~(1<<4);//配置更新
        while(!(RTC->CRL&(1<<5)));//等待RTC寄存器操作完成 
        
        RTC_Get();//设置完之后更新一下数据 	   
    }
    
    void RTC_Get(void)
    {
        static uint16_t daycnt=0;
        uint32_t timecount=0; 
        uint32_t temp=0;
        uint16_t temp1=0;	  
        timecount=RTC->CNTH;//得到计数器中的值(秒钟数)
        timecount<<=16;
        timecount+=RTC->CNTL;			 
    
        temp=timecount/86400;   //得到天数(秒钟数对应的)
        if(daycnt!=temp)//超过一天了
        {	  
            daycnt=temp;
            temp1=1970;	//从1970年开始
            while(temp>=365)
            {				 
                if(Is_Leap_Year(temp1))//是闰年
                {
                    if(temp>=366)temp-=366;//闰年的秒钟数
                    else break;  
                }
                else temp-=365;	  //平年 
                temp1++;  
            }   
            calendar.w_year=temp1;//得到年份
            temp1=0;
            while(temp>=28)//超过了一个月
            {
                if(Is_Leap_Year(calendar.w_year)&&temp1==1)//当年是不是闰年/2月份
                {
                    if(temp>=29)temp-=29;//闰年的秒钟数
                    else break; 
                }
                else 
                {
                    if(temp>=mon_table[temp1])temp-=mon_table[temp1];//平年
                    else break;
                }
                temp1++;  
            }
            calendar.w_month=temp1+1;	//得到月份
            calendar.w_date=temp+1;  	//得到日期 
        }
        temp=timecount%86400;     		//得到秒钟数   	   
        calendar.hour=temp/3600;     	//小时
        calendar.min=(temp%3600)/60; 	//分钟	
        calendar.sec=(temp%3600)%60; 	//秒钟
        calendar.week=RTC_Get_Week(calendar.w_year,calendar.w_month,calendar.w_date);//获取星期   
    }	
    
    //判断是否是闰年函数
    //月份   1  2  3  4  5  6  7  8  9  10 11 12
    //闰年   31 29 31 30 31 30 31 31 30 31 30 31
    //非闰年 31 28 31 30 31 30 31 31 30 31 30 31
    //year:年份
    //返回值:该年份是不是闰年.1,是.0,不是
    uint8_t Is_Leap_Year(uint16_t year)
    {			  
        if(year%4==0) //必须能被4整除
        { 
            if(year%100==0) 
            { 
                if(year%400==0)return 1;//如果以00结尾,还要能被400整除 	   
                else return 0;   
            }else return 1;   
        }else return 0;	
    }	
    
    //获得现在是星期几
    //功能描述:输入公历日期得到星期(只允许1901-2099年)
    //year,month,day：公历年月日 
    //返回值：星期号																						 
    uint8_t RTC_Get_Week(uint16_t year,uint8_t month,uint8_t day)
    {	
        uint16_t temp2;
        uint8_t yearH,yearL;
        
        yearH=year/100;	yearL=year%100; 
        // 如果为21世纪,年份数加100  
        if (yearH>19)yearL+=100;
        // 所过闰年数只算1900年之后的  
        temp2=yearL+yearL/4;
        temp2=temp2%7; 
        temp2=temp2+day+table_week[month-1];
        if (yearL%4==0&&month<3)temp2--;
        return(temp2%7);
    }
    ```
- 2.2 UnixRtc.h
    ``` c
    #ifndef __UNXINRTC_H
    #define __UNXINRTC_H


    typedef struct 
    {
        __IO uint8_t hour;
        __IO uint8_t min;
        __IO uint8_t sec;			
        //公历日月年周
        __IO uint16_t w_year;
        __IO uint8_t  w_month;
        __IO uint8_t  w_date;
        __IO uint8_t  week;	
    }_calendar_obj;	
    extern _calendar_obj calendar;				//日历结构体

    void rtc_init_user(void);
    void RTC_Set(uint16_t syear,uint8_t smon,uint8_t sday,uint8_t hour,uint8_t min,uint8_t sec);
    void RTC_Get(void);
    uint8_t Is_Leap_Year(uint16_t year);
    uint8_t RTC_Get_Week(uint16_t year,uint8_t month,uint8_t day);

    #endif
    ```