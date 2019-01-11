

# chat server 异步消息处理

## 现有状况

...
## 设计目标
 1. 异步处理chat server 一些耗时的消息处理流程，解耦chat server 和一些功能的依赖。
 2. 通过异步处理提升chat server 的性能。

## 总体思路
  ![mq](mq.png)
  Chat Server 一些同步的消息处理，将采用 service broker 进行异步解耦, consumer service 及时的消费service broker 中数据，完成后续处理流程。
  1. 消息处理流程：chat server 推送事件，将事件消息写入 service broker event queue事件队列中，Distributor service 负责将这些等待处理的事件信息推送到订阅了这些事件信息的 data queue 中，以供具体的消费者consumer去消费这些数据。
  2. 事件队列和订阅了这些事件的具体数据队列data queue 之间的关系存放到数据库中，Distributor service 定时更新关系数据。
  3. Remote Event MQ Server 为远程事件队列服务器，Main Event MQ Server down掉以后事件将会发送到上面，同时也作为副服务器的事件队列服务器。
  4. Remote Event MQ Server 上的事件数据，最终会被主服务器的Distribute Service 分发服务把事件分派到关注这些事件的真实数据队列中data MQ Server。
  5. 最终consumer service 将会从data MQ server 中 拿出这些数据，消费掉。





 

## Events

  
 |Event Name | Queues  | Description |    
  | - | - | :-: | :-: | - | 
  | [Chat.Ended](#Chat.Ended) | 1. Chat.Ended.Persistence    2. Chat.Ended.Email 3. Chat.Ended.Ticket  4. Chat.Ended.Webhook 5. Chat.Ended.Salesforce 6. Chat.Ended.Zendesk |   |
  | [OfflineMessage.Submit](#OfflineMessage.Submit) | 1. OfflineMessage.Submit.Persistence 2. OfflineMessage.Submit.Email 3. OfflineMessage.Submit.Ticket 4. OfflineMessage.Submit.Webhook 5. OfflineMessage.Submit.Salesforce 6. OfflineMessage.Submit.Zendesk   |   |
  | [Visitor.Rating](#Visitor.Rating) | Visitor.Rating  |  |
  | [Agent.Wrapup](#Agent.Wrapup) | Agent.Wrapup  |   |
  | [CannedMessage.UseLog](#CannedMessage.UseLog) |  CannedMessage.UseLog |   |
  | [PrivateMessage.Log](#PrivateMessage.Log) |  PrivateMessage.Log |   |
  | [ChatQueue.Log](#ChatQueue.Log) |  ChatQueue.Log |   |
  | [AgentStatus.Log](#AgentStatus.Log) |  AgentStatus.Log |   |
  | [AutoInvitation.Log](#AutoInvitation.Log) |  AutoInvitation.Log |   |
  | [ManualInvitation.Log](#ManualInvitation.Log) |  ManualInvitation.Log |   |
  | [Visit.Log](#Visit.Log) |  Visit.Log |   |
  | [Conversion.Log](#Conversion.Log) |  Conversion.Log |   |
  | [Agent.SavePreference](#Agent.SavePreference) |  Agent.SavePreference |   |
  | [Agent.Ban](#Agent.Ban) |  Agent.Ban |   |

  
###  Chat.Ended

#### Queue
1. Chat.Ended.Persistence
2. Chat.Ended.Email
3. Chat.Ended.Ticket
4. Chat.Ended.Webhook
5. Chat.Ended.Salesforce
6. Chat.Ended.Zendesk


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
      public string content{get;set;}
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
      public int VisitorMessageNum{get;set;}
      public int AgentMessageNum{get;set;}
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


###  OfflineMessage.Submit 

#### Queue
1. OfflineMessage.Submit.Persistence
2. OfflineMessage.Submit.Email
3. OfflineMessage.Submit.Ticket
4. OfflineMessage.Submit.Webhook
5. OfflineMessage.Submit.Salesforce
6. OfflineMessage.Submit.Zendesk


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
    public bool IfEnableIntegration{get;set;}
    public string Title{get;set;}
    public string Content{get;set;}
 }

 

```



###  Visitor.Rating  

#### Queue
Visitor.Rating

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






###  Agent.Wrapup  


#### Queue
Agent.Wrapup

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




###  CannedMessage.UseLog  

#### Queue
CannedMessage.UseLog


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

###  PrivateMessage.Log  

#### Queue
PrivateMessage.Log


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



###  ChatQueue.Log  

#### Queue
ChatQueue.Log


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



###  AgentStatus.Log  

#### Queue
AgentStatus.Log


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




###  AutoInvitation.Log  

#### Queue
AutoInvitation.Log


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
ManualInvitation.Log


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




###  Visit.Log

#### Queue
Visit.Log


#### Data Struct
```c#
    public class VisitLog
    {
     public int SiteId{get;set;}
     public int CampainId{get;set;}
     public int VisitCount{get;set;}

    }
 
```




###  Conversion.Log 

#### Queue
Conversion.Log


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
  public bool HasAddToDatabase{get;set;}
  public int CurrentChatId{get;set;}
  public int CurrentDepartmentId{get;set;}
  public int ConversionAccelerateType{get;set;}//0 firstChat 1 lastChat

}
 
```






###  Agent.SavePreference

#### Queue
Agent.SavePreference


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





###  Agent.Ban 

#### Queue
Agent.Ban


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


```
 