# Change-Monitoring-
Created to get notification when the Case Assignment Rule Updates
public class changeMonitor{
    List<Config_Change_Monitor__c> cmList = new List<Config_Change_Monitor__c>();
    Set<String> orgName = new Set<String>();

    public changeMonitor(){
        cmList = [SELECT Component_Name__c,Id,isActive__c,LastViewedDate,Last_Verified_DateTime__c,Name,Object_Name__c,Org_Name__c,OwnerId,Subscriber_Emails__c FROM Config_Change_Monitor__c where isActive__c = True];
            for(Config_Change_Monitor__c changeRec: cmList){
                orgName.add(changeRec.Org_Name__c);
            }
            
            for(String Orgs: orgName){
                HttpResponse res = auditTrailRequest(Orgs);
                List<auditTrailWrapper> auditList = responseParser(res);
                List<Config_Change_Monitor__c> cmTempList = new List<Config_Change_Monitor__c>();
                    for(Config_Change_Monitor__c allChangeRecs: cmList){
                        if(Orgs == allChangeRecs.Org_Name__c){
                        cmTempList.add(allChangeRecs);
                        }
                    }
                pairMonitornLogs(cmTempList, auditList);
            }
        updateVerifiedDateTime(cmList);
    }
    
    
    
    Public HttpResponse auditTrailRequest(String orgDA){
    String auditTrailURL = '/services/data/v36.0/query/?q=SELECT+Action,CreatedBy.Name,CreatedBy.Email,CreatedDate,DelegateUser,Display,Field1,Field2,Field3,Field4,Field5,Id,Section+FROM+SetupAuditTrail+WHERE+Action=\'caseassrule\'+AND+CreatedDate=Today';
        Http h = new Http();
        HttpRequest req = new HttpRequest();
        HttpResponse res= new HttpResponse();
        String endUrl = 'callout:' + orgDA + auditTrailURL;
        req.setEndpoint(endUrl);
        req.setMethod('GET');
        req.setTimeout(60000);
        req.setHeader('Content-Type', 'application/json');  
        res.setStatusCode(200);
        res = h.send(req); 
    
   return res;
   }
   
   Public List<auditTrailWrapper> responseParser(HttpResponse response){
   
   Map<String, String> outMap = new Map<String, String>();
   List<auditTrailWrapper> auditTrailList = new List<auditTrailWrapper>();
   String Action;
   String Email;
   String resRec;
   System.JSONToken token;
   
    if(response.getStatusCode() == 200)
           {
               System.debug('Response--------------------'+response.getBody());
                JSONParser parser = JSON.createParser(response.getBody());   
                    while((token = parser.nextToken()) != null) {
                           if ((token = parser.getCurrentToken()) != JSONToken.END_OBJECT) {
                           String text = parser.getText();
                           
                              if (token == JSONToken.FIELD_Name && text == 'Action') {
                                token=parser.nextToken();
                                Action = parser.getText();
                                resRec = Action;
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'Name') {
                                token=parser.nextToken();
                                resRec = resRec + ','  + parser.getText();
                              }
                           
                              if (token == JSONToken.FIELD_Name && text == 'Email') {
                                token=parser.nextToken();
                                Email = parser.getText();
                                resRec = resRec + ','  + parser.getText();
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'CreatedDate') {
                                token=parser.nextToken();
                                resRec = resRec + ','  + parser.getText();
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'Display') {
                                token=parser.nextToken();
                                resRec = resRec + ','  + parser.getText();
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'Field1') {
                                token=parser.nextToken();
                                resRec = resRec + ','  + parser.getText();
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'Field2') {
                                token=parser.nextToken();
                                resRec = resRec + ','  + parser.getText();
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'Field3') {
                                token=parser.nextToken();
                                resRec = resRec + ','  + parser.getText();
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'Field4') {
                                token=parser.nextToken();
                                resRec = resRec + ',' + parser.getText();
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'Field5') {
                                token=parser.nextToken();
                                resRec = resRec + ',' + parser.getText();
                              }
                              
                              if (token == JSONToken.FIELD_Name && text == 'Section') {
                                token=parser.nextToken();
                                resRec = resRec + ',' + parser.getText();
                                auditTrailWrapper audRec = new auditTrailWrapper(resRec);
                                auditTrailList.add(audRec);
                              }
                             
                           
                           }
                    outMap.put(Email, Action);       
            }
           }
           return auditTrailList;
    }
    
    public void pairMonitornLogs(List<Config_Change_Monitor__c> monitorList, List<auditTrailWrapper> trailList){
        Integer assignmentCount=0;
        String componentName;
        String objectName;
        String orgsName;
        String ChangesList = '';
        List<String> toList = new List<String>();
        Set<String> modifiers = new Set<String>();
        System.debug('Records Sub List---------' + monitorList);
        System.debug('AuditTrail List---------' + trailList);
        
        for(Config_Change_Monitor__c c: monitorList){
            List<auditTrailWrapper> assignementList;
            orgsName = c.Org_Name__c;
                    toList.addAll(c.Subscriber_Emails__c.split(';'));
                    System.debug('1st Loop' + c);
            for(auditTrailWrapper a: trailList){
                System.debug('2nd Loop' + a);
                if(a.ModifiedDateTime >= c.Last_Verified_DateTime__c){
                System.debug('1st Condition');
                if(c.Component_Name__c.contains('AssignmentRule') && a.Action == 'caseassrule' && c.Object_Name__c.contains(a.ObjectName)){
                    System.debug('2nd Condition');
                    assignmentCount++;
                    componentName = 'AssignmentRule';
                    objectName = a.ObjectName;
                    modifiers.add(a.Modifier);
                    ChangesList = ChangesList + '<br>' + a.Modifier + ' - ' + a.Display + ' - ' + (a.ModifiedDateTime-(1/3.0)) + '\n';
                    toList.add(a.ModifierEmail); 
                }
                }
                
            }
            System.debug('assignmentCount assignmentCount '+ assignmentCount);
            if(assignmentCount > 0){
            String modifierNames = modifierName(modifiers);
            String emailsBody = emailTemp(componentName, objectName, ChangesList, modifierNames);
            System.debug('AAAAAAAAAAA' + emailsBody);
            String sub = 'Case Assignment Rules are updated in ' + orgsName;
            sendEMail(toList, emailsBody, sub);
            }
        }
    }
    
    public String emailTemp(String compType, String ObjName, String changeList, String modifiedName){
        String htmlEmailTemplate = '<html><body>Hello Everyone, <br><br>'+ compType + ' have been changed on the ' + ObjName +' object <br>Changes were made by <br>'+ changeList + '<br>' + '<br>' + modifiedName + ', Could  you please provide the details of what was the change made with the case#' +' Thank you</body>  </html>';    
    return htmlEmailTemplate;
    }
    
    public String modifierName(Set<String> modifier){
    String Names = '';
     for(String name: modifier){
     String[] oName = name.split(' ');
     if(Names == ''){
     Names = oName[0];
     }else{
     Names = Names + '/' + oName[0];
     }
     }
     return Names;
    }
    
    public void sendEMail(List<String> Toemail, String body, String subject){
        Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
        List<Messaging.SingleEmailMessage> mails = new List<Messaging.SingleEmailMessage>();
      
        mail.setToAddresses(Toemail);
        mail.setSubject(subject);
        mail.setHtmlBody(body);
        mail.setReplyTo('no-reply@salesforce.com');
        mail.setSenderDisplayName('Change Monitor');
        mails.add(mail);
        Messaging.sendEmail(mails);
    } 
                           
    public void updateVerifiedDateTime(List<Config_Change_Monitor__c> changeRecList){
        DateTime now = system.now();
        List<Config_Change_Monitor__c> upChangeList = new List<Config_Change_Monitor__c>();
            for(Config_Change_Monitor__c ccm: changeRecList){
            ccm.Last_Verified_DateTime__c = now;
            upChangeList.add(ccm);
            }
        update upChangeList;
    }
}
