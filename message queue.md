

# chat server 异步消息处理

## 现有状况

...
## 设计目标
 1. 异步处理chat server 一些耗时的消息处理流程，解耦chat server 和一些功能的依赖。
 2. 通过异步处理提升chat server 的性能。

## 总体思路
  ![mq](mq.png)
  Chat Server 一些同步的消息处理，将采用 service broker 进行异步解耦, consumer service 及时的消费service broker 中数据，完成后续处理流程。
  1. 消息处理流程：chat server 推送事件，将事件消息写入 service broker event queue Server 中的事件队列中， 存储过程负责将这些等待处理的事件信息写入到订阅了这些事件信息的 data queue 中（订阅关系配置在第2点中），以供具体的消费者consumer去消费这些数据。
  2. service broker event queue 和data queue 的关系配置在表 [t_chatserver_event](#t_chatserver_event)和 [t_chatserver_queue](#t_chatserver_queue)中。
  3. Remote  Service Broker Event Queue Server 为远程事件队列服务器，Main Service Broker Event Queue Server down掉以后事件将会发送到上面，同时也作为副服务器的事件队列服务器。
  4. Remote  Service Broker Event Queue Server 上的事件数据， 会通过service broker的配置将数据同步到主服务器上的data queue中。
  5. 最终consumer service 将会从data queue 中 拿出这些数据，消费掉，完成最后的流程处理。



## Events

  
 |Event Name |  Description |    
  | - | :-: |
  | [chat.queued](#chat.queued) |  chat 排队事件|   |
  | [chat.started](#chat.started) | chat 聊天开始事件  |   
  | [chat.visitor.replied](#chat.visitor.replied) | 访客回复消息事件  |   
  | [chat.agent.replied](#chat.agent.replied) |  坐席回复消息事件 |  
  | [chat.ended](#chat.ended) | 聊天结束事件 |   
  | [chat.wrapup.submitted](#chat.wrapup.submitted) |wrapup 提交事件 |   
  | [chat.rating.submitted](#chat.rating.submitted) | rating 提交事件|   
  | [visitor.landed](#visitor.landed) |访客登入 |  
  | [visitor.conversion.achieved](#visitor.conversion.achieved) |有效客户转换事件 |   
  | [ban.added](#ban.added) | 添加黑名单事件|   
  | [offlineMessage.submitted](#offlineMessage.submitted) |离线留言事件 |   
  | [agent.status.changed](#agent.status.changed) | 坐席状态改变事件|   
  | [agent.preference.changed](#agent.preference.changed) | 坐席个性化设置变更事件|   
  | [agentChat.replied](#agentChat.replied) |坐席私聊回复事件 |   
  | [cannedMessage.used](#cannedMessage.used) | 快捷信息使用事件 |   
  | [autoInvitation.sent](#autoInvitation.sent) | 自动邀请发送事件 |   
  | [autoInvitation.accepted](#autoInvitation.accepted) | 自动邀请接收事件 |   
  | [autoInvitation.refused](#autoInvitation.refused) | 自动邀请被拒绝事件 |  


## Service Broker 结构

### MessageType

 |MessageType Name | Validation  | 
  | - | :-: |
  | JsonType | None |


### Contract

 |Contract Name | Send by  | 
  | - | :-:|
  | GeneralContract | Any |

### Event Service And Queue Relationship


  |Event Name| Send Service  Name  | Send Service Binding Queue | Recive Service Name |Recive Service  Binding Queue |
  | - | - | :-: | :-: | :-: |
  |[chat.queued](#chat.queued)| Chat.Queued.SendService| Chat.Queued.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[chat.started](#chat.started)| Chat.Started.SendService| Chat.Started.SendQueue |WebHook.ReciveService|WebHookQueue|
  |[chat.ended](#chat.ended)| Chat.Ended.SendService| Chat.Ended.SendQueue |Chat.Ended.ReciveService|Chat.Ended.ReciveQueue|
  |[chat.wrapup.submitted](#chat.wrapup.submitted)| Chat.Wrapup.Submitted.SendService| Chat.Wrapup.Submitted.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[chat.rating.submitted](#chat.rating.submitted)| Chat.Rating.Submitted.SendService| Chat.Rating.Submitted.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[visitor.landed](#visitor.landed)| Visitor.Landed.Submitted.SendService| Visitor.Landed.Submitted.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[visitor.conversion.achieved](#visitor.conversion.achieved)| Visitor.Conversion.Achieved.SendService| Visitor.Conversion.Achieved.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[ban.added](#ban.added)| Ban.Added.SendService| Ban.Added.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[offlineMessage.submitted](#offlineMessage.submitted)| OfflineMessage.Submitted.SendService| OfflineMessage.Submitted.SendQueue |OfflineMessage.Submitted.ReciveService|OfflineMessage.Submitted.ReciveQueue|
  |[agent.status.changed](#agent.status.changed)| Agent.Status.Changed.SendService| Agent.Status.Changed.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[agent.preference.changed](#agent.preference.changed)| Agent.Preference.Changed.SendService| Agent.Preference.Changed.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[agentChat.replied](#agentChat.replied)| AgentChat.Replied.SendService| AgentChat.Replied.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[cannedMessage.used](#cannedMessage.used)| CannedMessage.Used.SendService| CannedMessage.Used.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[autoInvitation.sent](#autoInvitation.sent)| AutoInvitation.Sent.SendService| AutoInvitation.Sent.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[autoInvitation.accepted](#autoInvitation.accepted)| AutoInvitation.Accepted.SendService| AutoInvitation.Accepted.SendQueue |Persistence.ReciveService|PersistenceQueue|
  |[autoInvitation.refused](#autoInvitation.refused)| AutoInvitation.Refused.SendService| AutoInvitation.Refused.SendQueue |Persistence.ReciveService|PersistenceQueue|


### Consume Queue Service 

  | Send Service  Name  | Send Service Binding Queue | Recive Service Name |Recive Service  Binding Queue |
  | - | :-: | :-: | :-: |
  | Persistence.SendService| Persistence.SendQueue |Persistence.ReciveService|PersistenceQueue|
  | Email.SendService| Email.SendQueue |Email.ReciveService|EmailQueue|
  | Ticket.SendService| Ticket.SendQueue |Ticket.ReciveService|TicketQueue|
  | Salesforce.SendService| Salesforce.SendQueue |Salesforce.ReciveService|SalesforceQueue|
  | Zendesk.SendService| Zendesk.SendQueue |Zendesk.ReciveService|ZendeskQueue|
  | WebHook.SendService| WebHook.SendQueue |WebHook.ReciveService|WebHookQueue|
  

###  Consume Queues

  | Queue  Name  | description |
  | - | :-: | 
  | PersistenceQueue| 持久化队列 |
  | EmailQueue| 邮件队列 |
  | TicketQueue| 工单队列 |
  | SalesforceQueue| salesforce 队列 |
  | ZendeskQueue| zendesk 队列  |
  | WebHookQueue|  WebHook队列  |

###  Error Service And Queue

  | Send Service  Name  | Send Service Binding Queue | Recive Service Name |Recive Service  Binding Queue |Event Name |
  | - | :-: | :-: | :-: |:-: |
  | Error.SendService| Error.SendQueue |Error.ReciveService|ErrorQueue|Error|

### Event And Queue Relationship Table

#### t_chatserver_event

| Column  Name  | Type | Nullable |Default |Version |Primary key|Remark|
  | - | :-: | :-: | :-: |:-: |:-: |:-: |
  | Id| int |no||1.0|true|事件id|
  | Name| nvarchar(256) |no|''|1.0|false|事件名称|
  | SendServiceName| nvarchar(256) |no|''|1.0|false|发送服务名称|
  | SendQueueName| nvarchar(256) |no|''|1.0|false|发送队列名称|
  | ReciveServiceName| nvarchar(256) |no|''|1.0|false|接收服务名称|
  | ReciveQueueName| nvarchar(256) |no|''|1.0|false|接收队列名称|


#### t_chatserver_queue

| Column  Name  | Type | Nullable |Default |Version |Primary key|Remark|
  | - | :-: | :-: | :-: |:-: |:-: |:-: |
  | EventId| int |no||1.0|false|t_chatserver_event.id 外键|
  | Name| nvarchar(256) |no|''|1.0|false|队列名称|



## Event Produce  And Consume
  ![imq](mqinterface.png)

### Initialize 

应用程序启动时，初始化 EventFactory 配置,EventFactory 内部实现，将自动心跳检查配置的服务器状态，自动切换服务器。

```c# 


public void Initialize()
{
   ServerConfig serverConfig=new ServerConfig();
   serverConfig.Servers.Add(new Server(){Ip=xxx,port=xxx,name=Master,Password=xxx,IsMaster=true,IsActive=true});//添加Server broker主服务器
     serverConfig.Servers.Add(new Server(){Ip=xxx,port=xxx,name=Slave,Password=xxx,IsMaster=true,IsActive=true});//添加Server broker副服务器
   EventFactory.Initialize(serverConfig);

}
 

```



### Produce

```c# 

 
public class EventData
{

  public EventType Type{get;set;} 

 public object Data{get;set;}

}


public class EventProducer
{
     
    public bool ChatEnd(Chat chat)
    {
   
      IEventContext context=  EventFactory.Open();
      EventData data=new EventData();
      data.Type=EventType.chatEnded;
      data.Data=chat;
      string data=SerializeObject(queueData);
      return  context.Put(data);
    }
   
   public bool OfflineMessage(OfflineMessage offlineMessage)
   {
      IEventContext context=  EventFactory.Open();
       EventData data=new EventData();
       queueData.Type=EventType.offlineMessageSubmitted;
      queueData.Data=offlineMessage;
      string data=SerializeObject(queueData);
      return  context.Put("chat.ended",data);

   }
   ...
   ...
    string   SerializeObject(object data)
    {

       return    JsonConvert.SerializeObject(chat);   
    }   

}


```
 


###  Consumer

  ![imq](consume.png)
```c# 
  

 
 
 public void  Initialize()//消费者初始化，在程序启动的时候调用一次即可
 {
    
     int emailThreadHandleCount = 0;
     int.TryParse(ConfigurationManager.AppSettings["EmailThreadHandleCount"].ToString(),out emailThreadHandleCount);//邮件消费队列，每个线程处理数，实际消费线程数是该值的倍数。
     ConsumerManager.Run(new EmailConsumer(), emailThreadHandleCount);
     ...//在此加入其它消费队列实现。
 }

    public class ConsumerManager
    {

        private static Dictionary<string, ConsumerWrap> consumers = new Dictionary<string, ConsumerWrap>();

        public static void Run(IConsumer consumer,int everyThreadHandleCount)
        {
            if (!consumers.ContainsKey(consumer.QueueName))
                consumers.Add(consumer.QueueName, new ConsumerWrap(consumer, everyThreadHandleCount));
        }

    }



   public class ConsumerWrap
    {
        private ConcurrentBag<ConsumerThread> threads;
        private IConsumer consumer;
        private int everyThreadHandleCount;
        private int queueDataCount = 0;
        private Thread queryCountThread = null;
        public ConsumerWrap(IConsumer consumer, int everyThreadHandleCount)
        {
            this.consumer = consumer;
            this.everyThreadHandleCount = everyThreadHandleCount;
            this.threads = new ConcurrentBag<ConsumerThread>();
            this.UpdateConsumers(everyThreadHandleCount);
            this.queryCountThread = new Thread(QueryCount);
            this.queryCountThread.IsBackground = true;
            this.queryCountThread.Start();
        }

        private void UpdateConsumers(int queueDataCount)
        {
            int threadCount = (int)Math.Ceiling((Decimal)(queueDataCount / everyThreadHandleCount));

            int dif = threadCount - threads.Count;
            if (dif == 0) return;
            if (dif > 0)
            {
                for (int i = 0; i < dif; i++)
                {
                    ConsumerThread consumerThread = new ConsumerThread(consumer);
                    consumerThread.IsRunning = true;
                }
            }
            else
            {
                dif = System.Math.Abs(dif);
                for (int i = 0; i < dif; i++)
                {
                    ConsumerThread consumerThread = null;
                    if (threads.TryTake(out consumerThread))
                        consumerThread.IsRunning = false;
                }

            }
        }

        private void QueryCount()
        {
            while (true)
            {


                IEventContext context = null;
                try
                {
                    context = EventFactory.Open();
                    queueDataCount = context.GetCount(consumer.QueueName);
                    this.UpdateConsumers(queueDataCount);
                }
                catch (Exception ex)
                {

                }

                Thread.Sleep(1000);
            }
        }

    }


    public class ConsumerThread
    {

        public IConsumer Consumer { get; set; }
        private bool isRunning = false;
        private Thread thread;
        public bool IsRunning
        {
            get { return isRunning; }
            set
            {
                isRunning = value;
                if (isRunning)
                {
                    thread.Start();
                }
            }
        }

        public ConsumerThread(IConsumer consumer)
        {
            this.Consumer = consumer;

            thread = new Thread(excute);
            thread.IsBackground = true;

        }

        void excute()
        {
            while (isRunning)
            {
                Consumer.Consume();
                Thread.Sleep(50);
            }
            thread.Abort();
            thread = null;
        }

    }



public interface IConsumer
{
  string QueueName { get; }
  void Consume();

}


public class EmailConsumer: IConsumer
{

  public string QueueName { get { return "EmailQueue"; } }
 

  public  void Consume()
  {
       
      IEventContext context=  EventFactory.Open();
      try
      {
      string data= context.Get("EmailQueue");
      if(string.isnullorempty(data))continue;
      QueueData queueData=SerializeObject(data);
      if(queueData==null)
      {
        //添加到错误队列中
        context.Put("Error",data);
        context.Confirm();
        continue;
      }
      switch(queueData.Type)
      {
        ...
        case EventType.offlineMessageSubmitted:
              if(queueData.Data数据格式及内容校验==false)
              {
              //添加到错误队列中
              context.Put("Error",data);
              context.Confirm();
              //记录日志
              continue;
              }
          //发邮件代码
          ...
          Thread.Sleep(50);
        break;
        ...
      }
      
        
      ... 
      //完成持久化操作
      ...
      context.Confirm();
      } 
      catch(exception ex)
      {
        context.Roolback();
      }   
  }

  
}

```



## Event Details


###  chat.queued
#### Target Queue
  PersistenceQueue

#### Data Struct

```c#
    
public class ChatQueueLogs
{
 
  public int SiteId{get;set;}
  public List<ChatQueueLog> Messages{get;set;}
   
}


public class ChatQueueLog
{

  public int DepartmentId{get;set;}
  public int NumberOfQueue{get;set;}
  public DateTime LogTime{get;set;}
  
}
 
```



###  chat.started
#### Target Queue
WebHookQueue
#### Data Struct



###  chat.visitor.replied
#### Target Queue
 
#### Data Struct



###  chat.agent.replied
#### Target Queue
 
#### Data Struct


###  chat.ended
#### Target Queue
1. PersistenceQueue
2. EmailQueue
3. TicketQueue
4. SalesforceQueue
5. ZendeskQueue
6. WebHookQueue 
#### Data Struct

```c#
    public class Chat
    {
      public string ChatId{get;set;}
      public long  SessionId{get;set;}
      public int SiteId{get;set;}
      public List<int> AgentIds{get;set;}
      public DateTime  Start{get;set;}
      public DateTime End{get;set;}
      public string PreChatName{get;set;}
      public string PreChatCompany{get;set;}
      public string PreChatPhone{get;set;}
      public string PreChatEmail{get;set;}
      public string PreChatProductService{get;set;}
      public int PreChatDepartmentId{get.;set;}
      public string PreChatDepartmentName{get;set;}
      
      public string AgentComment{get;set;}
      public List<ChatMessage> Messages{get;set;}
      public int SocialMediaSource{get;set;}//0 None,1 Facebook,2 GooglePlus
      public string SocialMediaSource{get;set;}
      public string SocialProfileUrl{get;set;}
      public int ChatType{get;set;}// 0 AgentOnly ,1 ChatbotOnly,2 FromChatBotToAgent,3 Chatbot
      public bool IfAudioChatHappened{get;set;}
      public bool IfVideoChatHappened{get;set;}
      public int ChatSource{get;set;} //0 button ,1 autoInvitation ,2 manualInvitation
      public int  MissedChatStatus{get;set;} //0 Normal,1  AgentRefused,2 Missed,3 OfflineMessage
      public DateTime RequestTime{get;set;}
      public bool IfEnterQueue{get;set;}
      public double AgentAvgResponseTime{get;set;}
      public int LasMessageSendBy{get;set;}// -1 unknow ,0 visitor ,1 agent,2 system 
      public bool IfNewTicket{get;set;}
      public string TicketId{get;set;} 
      public string RequestPageTitle{get;set;}
      public string RequestPageUrl{get;set;}
   
      public int RatingGrade{get;set;}
      public string RatingComment{get;set;}
     
      public List<CustomField> CustomFields{get;set;}
      public List<CustomVariable> CustomVariable{get;set;}

      public List<ChatTransferLog> TransferLogs{get;set;}
      public List<Attachment> Attachments{get;set;}

      public Cobrowse Cobrowse{get;set;}
      public List<BotAction> BotActions{get;set;}
   
    }


   public class ChatMessage
   {
      public int Id{get;set;}
      public int ChatAction{get;set;}
      public int SenderType{get;set;} //0 visitor ,1 agent,2 system,3 chatbot
      public string SenderName{get;set;}
      public string Message{get;set;}
      public string translatedMessage{get;set;}
      public string Attachement{get;set;}
      public DateTime Time{get;set;}

   }

    public class Visitor
    {
      public string Id{get;set;}
      public string Country{get;set;}
      public string City{get;set;}
      public int TimeZone{get;set;}
      public string Language{get;set;}
      public string LastName{get;set;}
      public string LastEmail{get;set;}
      public int RelatedType{get;set;} //o None,1 Agent,2 User,3 Contract,4 EmailAddress,5 Cookie,6 Visitor
      public long RelatedId{get;set;}
      public List<int> segmentIds{get;set;}

    }

    public class Salesforce
    {
       public int PurposeChatEnd{get;set;} //0 createCase 1 AttachCaseToContact ,2 AttachTaskToContact,3 AttachTaskToLead ,4 CreateContactAndAttachCase ,5 CreateContactAndAttachTask,6 CreateLeadAndAttachTask ,7 UpdateNullContactFieldsAnd AttachCase,8 UpdateAllContactFieldsAndAttachCase,9 UpdateNullContactFieldsAndAttachTask,10 UpdateAllContactFiledsAndAttachTask
       public List<SalesforceFieldValue> Cases{get;set;}
       public List<SalesforceFieldValue> Tasks{get;set;}
       public List<SalesforceFieldValue> Leads{get;set;}
       public List<SalesforceFieldValue> CreateContact{get;set;}
       public List<SalesforceFieldValue> UpdateContact{get;set;}

    }

    public class SalesforceFieldValue
    {
     public string Name{get;set;}
     public string value{get;set;}
     public string DisplayValue{get;set;}

    }
 
    public class ChatTransferLog
    {
      public DateTime Time{get;set;}
      public int TransferType{get;set;} //0 agent,1 department
      public int FromId{get;set;}
      public int ToId{get;set;}
      public int AgentId{get;set;}
      public bool IfSuccess{get;set;}
    }


    public class Attachment
    {
        public string Uid{get;set;}
        public string FilePath{get;set;}
        public string Url{get;set;}
        public bool IsNoteFile{get;set;}

    }

    public class Cobrowse
    {
       public DateTime Start{get;set;}
       public DateTime End{get;set;}
       public int Status{get;set;}//0 Inited,1 Requested ,2 Accepted,3 Completed

    }


    public class BotAction
    {
      public string Guid{get;set;}
      public int ChatId{get;set;}
      public string OriginalQuestion{get;set;}
      public int StandardQuestionId{get;set;}
      public int  BotAnswerType{get;set;} //0 HighConfidenceAnswer,1  PossibleAnswer ,2 NoAnswer
      public int RateType{get;set;}
      public bool IfDeleted{get;set;}
      public DateTime Time{get;set;}
      public bool IsAdd{get;set;}


    }

```

###  chat.wrapup.submitted
#### Queue
 PersistenceQueue
#### Data Struct

```c#
    
public class Wrapup
{
 
  public int SiteId{get;set;}
  public int AgentId{get;set;}
  public int ChatId{get;set;}
  public int CategoryId{get;set;}
  public List<int> CategoryList{get;set;}
  public string Comment{get;set;}
  public DateTime SubmitTime{get;set;}
  public List<CustomField> CustomFields{get;set;}

}
 
```


###  chat.rating.submitted
#### Queue
 PersistenceQueue
#### Data Struct

```c#
    
public class Rating
{
  public int SiteId{get;set;}
  public int ChatId{get;set;}
  public string ChatGuid{get;set;}
  public int ratingGrade{get;set;}
  public string ratingComment{get;set;}
  public List<CustomField> CustomFields{get;set;}
  public List<CustomVariable> CustomVariables{get;set;}
}
 
```

###  visitor.landed
#### Queue
 PersistenceQueue
#### Data Struct

```c#
public class VisitLog
{
  public int SiteId{get;set;}
  public int CampainId{get;set;}
  public int VisitCount{get;set;}

}
 
```


###  visitor.conversion.achieved
#### Queue
 PersistenceQueue
#### Data Struct

```c#
public class ConversionLogs
{

  public int SiteId{get;set;}
  public List<ConversionLog> Logs{get;set;}
}


public class ConversionLog
{

 
 public int Id {get;set;}
  public int ConversionId{get;set;}
  public string ConversionName{get;set;}
  public long VisitorId{get;set;}
  public long CookIdenitityId{get;set;}
  public int ChatId{get;set;}
  public int DepartmentId{get;set;}
  public int RelatedOperatorId{get;set;}
  public string AppendInfo{get;set;}
  public double ConversionValue{get;set;}
  public DateTime CreateTime{get;set;}
  publ ic bool HasAddToDatabase{get;set;}
  public int CurrentChatId{get;set;}
  public int CurrentDepartmentId{get;set;}
  public int ConversionAccelerateType{get;set;}//0 firstChat 1 lastChat

}
 
```

### ban.added
#### Queue
 PersistenceQueue
#### Data Struct

```c#
public class Ban
{

  public int SiteId{get;set;}
  public int  AgentId{get;set;}
  public int BanType{get;set;}//0 visitorId ,1 ip, 2 iparrange
  public long IpFromOrVisitorId{get;set;}
  public long IpTo{get;set;}
  public string Comment{get;set;}
  public int OperatorId{get;set;}
}


 
 
```



###  offlineMessage.submitted
#### Queue
1. PersistenceQueue
2. EmailQueue
3. TicketQueue
4. SalesforceQueue
5. ZendeskQueue
6. WebHookQueue 
#### Data Struct

```c#
 public class OfflineMessage
 {
    public string visitorId{get;set;}
    public string VisitorName{get;set;}
    public string VisitorEmail{get;set;}
    public string VisitorCompany{get;set;}
    public string VisitorPhone{get;set;}
    public string City{get;set;}
    public string Country{get;set;}
    public string TimeZone{get;set;}
    public string Language{get;set;}
    public int SiteId{get;set;}
    public long SessionId{get;set;} 
    public string SourceChatId{get;set;}
    public int CampainId{get;set;}
    public int TicketId{get;set;}
    public int RouteToId{get;set;}
    public int RouteToType{get;set;}//0 site,1 department  ,2 operator ,3 empty,4  offlinemessage 
    public int ChatSource{get;set;} //0 chat Button,1 AutoInvitation,2 ManuallyInvitation
    public string RequestPageTitle{get;set;}
    public string RequestPageUrl{get;set;}
    public int AutoInvitationId{get;set;}
    public string ProductService{get;set;}
    public List<int> Segments{get;set;}
    public string AttachmentName{get;set;}
    public byte[] AttachmentContent{get;set;}
    public List<CustomField> CustomFields{get;set;}
    public List<CustomVariable> CustomVariables{get;set;}
    public string Title{get;set;}
    public string Content{get;set;}
 }


 
```

###  agent.status.changed
#### Queue
 PersistenceQueue
#### Data Struct

```c#
    
public class AgentStatusLog
{
 
  public int SiteId{get;set;}
  public int AgentId{get;set;}
  public int Stauts{get;set;}
  public DateTime Time{get;set;}
}

 
 
```


###  agent.preference.changed
#### Queue
PersistenceQueue
#### Data Struct

```c#
public class SavePreference
{

  public int SiteId{get;set;}
  public int  AgentId{get;set;}
  public List<Column> Columns{get;set;}
}


public class Column
{
   public int EnumColumn{get;set;}
   public bool IfVisible{get;set;}
   public int Width{get;set;}
   public int CustomVariableId{get;set;}
}

 
```





### agentChat.replied
#### Queue
PersistenceQueue
#### Data Struct


```c#
    
public class PrivateMessageLogs
{
 
  public int SiteId{get;set;}
  public List<PrivateMessage> Messages{get;set;}
   
}


public class PrivateMessageLog
{

  public int FromAgentId{get;set;}
  public int ToAgentId{get;set;}
  public DateTime Time{get;set;}
  public string Content{get;set;}
  public string AttachmentGuid{get;set;}
}
 
```



###  cannedMessage.used
#### Queue
PersistenceQueue
#### Data Struct

```c#
    
public class CannedMessage
{
 
  public int SiteId{get;set;}
  public int ChatId{get;set;}
  public int AgentId{get;set;}
  public int CannedMessgeId{get;set;}
  public DateTime Time{get;set;}
   
}

 
```


###  autoInvitation.sent
#### Queue
PersistenceQueue
#### Data Struct

```c#
    
public class AutoInvitationLogs
{
  public int SiteId{get;set;}
  public List<AutoInvitationLog> Logs{get;set;}

}

    
public class AutoInvitationLog
{
 
 
  public int CampainId{get;set;}
  public int InvitaionId{get;set;}
  public int SetNumber{get;set;}
  public int AcceptNumber{get;set;}
  public int RefuseNumber{get;set;}
  public DateTime Time{get;set;}
 
}


```


###  autoInvitation.accepted
#### Queue
PersistenceQueue
#### Data Struct

```c#
    
public class AutoInvitationLogs
{
  public int SiteId{get;set;}
  public List<AutoInvitationLog> Logs{get;set;}

}

    
public class AutoInvitationLog
{
 
 
  public int CampainId{get;set;}
  public int InvitaionId{get;set;}
  public int SetNumber{get;set;}
  public int AcceptNumber{get;set;}
  public int RefuseNumber{get;set;}
  public DateTime Time{get;set;}
 
}


```


###  autoInvitation.refused
#### Queue
PersistenceQueue
#### Data Struct

```c#
    
public class AutoInvitationLogs
{
  public int SiteId{get;set;}
  public List<AutoInvitationLog> Logs{get;set;}

}

    
public class AutoInvitationLog
{
 
 
  public int CampainId{get;set;}
  public int InvitaionId{get;set;}
  public int SetNumber{get;set;}
  public int AcceptNumber{get;set;}
  public int RefuseNumber{get;set;}
  public DateTime Time{get;set;}
 
}


```

###  ManualInvitation.Log

#### Queue
PersistenceQueue


#### Data Struct
```c#
    
public class ManualInvitationLogs
{
 
 public int SiteId{get;set;}
 public List<ManualInvitationLog> Logs{get;set;}
 
}


public class  ManualInvitationLog
{
 public int AgentId{get;set;}

  public int Status{get;set;} // 0 Miss,1 Accept ,2 Refuse

  public int TranscriptType{get;set;} //0 chat 1 offlineMessage,2 navigation

  public int TranscriptId{get;set;}

  public string Message{get;set;}

}
```

 



## Common Data Struct
 ```c#
public class  CustomField
{
  public int Id{get;set;}
  public string Name{get;set;}
  public string Value{get;set;}

}

public class CustomVariable
{
   
   public int Id{get;set;}
   public string Name{get;set;}
   public string value{get;set;}
   public string url{get;set;}
}

public class EventData
{

  public EventType Type{get;set;} //1.chat.queued 2.chat.started 3.chat.visitor.replied 4.chat.agent.replied 5.chat.ended 6.chat.wrapup.submitted 7. chat.rating.submitted 8.visitor.landed 9.visitor.conversion.achieved 10.ban.added 11.offlineMessage.submitted 12.agent.status.changed 13.agent.preference.changed 14.agentChat.replied 15.cannedMessage.used 16.autoInvitation.sent 17.autoInvitation.accepted 18.autoInvitation.accepted

 public object Data{get;set;}


}

public enum EventType
{
none=0,
chatQueued =1,
chatStarted=2 ,
chatVisitorReplied=3 ,
chatAgentReplied =4,
chatEnded =5,
chatWrapupSubmitted=6 ,
chatRatingSubmitted =7,
visitorLanded =8,
visitorConversionAchieved =9,
banAdded =10,
offlineMessageSubmitted=11,
agentStatusChanged =12,
agentPreferenceChanged=13 ,
agentChatReplied =14,
cannedMessageUsed =15,
autoInvitationSent=16 ,
autoInvitationAccepted=17 ,
autoInvitationAccepted=18
   
}


```
 