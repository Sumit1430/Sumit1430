public without sharing class SeedStudentApplicationController {

    public static String yearValue = String.valueOf((Date.today()).year());
   
@AuraEnabled(cacheable=true)
    public static StudentApplicationWrapper getStudentApplicationDetails(String fetchDetails){
        FetchDetails details = new FetchDetails();
        if(String.isNotBlank(fetchDetails))
        details = (FetchDetails) JSON.deserialize(fetchDetails, FetchDetails.class);
        StudentApplicationWrapper wrapper = new StudentApplicationWrapper();
        wrapper.contactDetails = getContactDetails();
        wrapper.listOfSteps = getSteps();
       
        Decimal latitude = wrapper.contactDetails?.Mailing_Latitude__c;
        Decimal longitude = wrapper.contactDetails?.Mailing_Longitude__c;
       
        if(details.fetchEssayQuestions){
            wrapper.essayQuestions = getEssayQuestions();
        }
       
        if(details.fetchListOfSites && latitude != null && longitude != null){
            wrapper.listSiteDetails = getProjectDetails(latitude, longitude).values();
        }
       
        if(details.fetchSiteDetails && latitude != null && longitude!= null){
            Map<Id, SiteDetailsWrapper> mapSiteIdVsWarpper = getProjectDetails(latitude, longitude);
            if(mapSiteIdVsWarpper.containsKey(wrapper.contactDetails.Application_Forms1__r[0].Sites__c)){
                wrapper.siteDetails = mapSiteIdVsWarpper.get(wrapper.contactDetails.Application_Forms1__r[0].Sites__c);
            }
        }
        return wrapper;
    }
   
    @AuraEnabled
    public static String updateContact(String contactDetails, String applicationDetails){
        Savepoint sp = Database.setSavepoint();
        try{
            List<sObject> listOfsObjectToUpdate = new List<sObject>();
            if(String.isNotBlank(contactDetails)){
                Contact con = (Contact) JSON.deserialize(contactDetails, Contact.class);
            listOfsObjectToUpdate.add(con);
            }
            if(String.isNotBlank(applicationDetails)){
                Application_Form__c application = (Application_Form__c) JSON.deserialize(applicationDetails, Application_Form__c.class);
                application.Year_of_Application__c = yearValue;
                listOfsObjectToUpdate.add(application);
            }
            if(!listOfsObjectToUpdate.isEmpty()){
upsert listOfsObjectToUpdate;                
            }
            return 'success';
        } catch(Exception ex){
            database.rollback(sp);
            return ex.getMessage();
        }
    }
   
    public static List<Seed_Student_Application_Step__mdt>  getSteps(){
        String query = 'SELECT MasterLabel, Page_Name__c, Order__c FROM Seed_Student_Application_Step__mdt ORDER BY Order__c ASC';
        return Database.query(query);
    }
   
    public static SeedCoordinatorTimeline__mdt getEssayQuestions(){
        String query = 'SELECT Essay_1__c, Essay_2__c, Essay_3__c, Student_Application_Start_Date__c, ';
        query += 'Student_Application_End_Date__c  FROM SeedCoordinatorTimeline__mdt WHERE ';
        query += 'MasterLabel = :yearValue LIMIT 1';
        SeedCoordinatorTimeline__mdt essayQuestions = Database.query(query);
        return essayQuestions;
    }
   
    public static Contact getContactDetails(){
        String userId = UserInfo.getUserId();
        String query = 'SELECT Id, Salutation, FirstName, LastName, Suffix__c, Birthdate, SEED_Above_18_Years__c,';
        query += 'SEED_Primary_Email__c, Phone, Address_Line_1__c, Address_Line_2__c, City__c, State__c, Zip_Postal_Code__c,';
        query += 'SEED_Gender__c, SEED_Preferred_Pronouns__c, SEED_Student_Ethnicity__c, SEED_Race__c, SEED_Language_Spoken__c, ';
        query += 'SEED_Parent_Prefix__c, SEED_Parents_Guardian_Name__c, SEED_Parent_Last_Name__c, SEED_Parents_Guardian_Suffix__c, ';
        query += 'Parent_Email_Address_Present_Or_Not__c, Parent_Address_Same_As_Student__c, ';
        query += 'SEED_Parent_Guardian_Email__c, SEED_Parents_Guardian_Phone_Number__c, SEED_Parent_s_Address_Street_Name__c, ';
        query += 'SEED_Parent_Address_ApartmentUnit_Number__c, SEED_Parent_s_Address_City__c, SEED_Parent_s_Address_State__c, ';
        query += 'SEED_Parent_s_Address_ZIP_Code__c, SEED_Household_Family_Size__c, SEED_Total_Family_Income_from_last_2_yrs__c, ';
        query += 'SEED_Graduation_Year__c, SEED_Have_Transportation__c, SEED_Mode_of_Transportation__c, ';
        query += 'Mailing_Latitude__c, Mailing_Longitude__c, Email,';
        query += '(SELECT Id, Student__c, SEED_Household_Member_Applying__c, Household_Member_With_Same_Parent__c, ';
        query += 'SEED_High_School_Name__c, SEED_Type_of_School__c, SEED_Experience__c, SEED_Last_day_of_school__c, ';
        query += 'SEED_First_day_of_school__c, SEED_After_Graduation_Plan__c, SEED_STEM_Activites__c, ';
        query += 'SEED_STEM_Activities_Other__c, SEED_Extracurriculars__c, SEED_Extracurriculars_Other__c, ';
        query += 'SEED_GPA__c, SEED_Max_GPA__c, SEED_Essay_1__c, SEED_Essay_2__c, SEED_Essay_3__c, ';
        query += 'SEED_Summer_Activities__c, SEED_Summer_Activity_Specifics__c, SEED_Disability__c, Sites__c, ';
        query += 'Sites__r.Name, Student_Application_Last_Visited_Page__c, SEED_HS_Grad_Year__c, SEED_After_Graduation_Plan_Other__c ';
        query += 'FROM Application_Forms1__r WHERE Year_of_Application__c = :yearValue ORDER BY CreatedDate ASC) ';
        query += 'FROM Contact WHERE Id IN (SELECT ContactId FROM USER WHERE Id = :userId)';
        Contact con = Database.query(query);
        return con;
    }
   
    public static List<SiteDetailsWrapper> getSiteDetails(Decimal latitude, Decimal longitude){
        Map<Id, SiteDetailsWrapper> mapSiteIdVsWarpper = getProjectDetails(latitude, longitude);
        return mapSiteIdVsWarpper.values();
    }
   
    private static Map<Id, SiteDetailsWrapper> getProjectDetails(Decimal latitude, Decimal longitude) {
        latitude =15;
        longitude =90;
        
        Map<Id,SEED_Project__c> mapProjIdVsDetails = new Map<Id,SEED_Project__c>
        ([SELECT Id, Sites__c, Sites__r.Name, Institution_Address_Line_1__c,
                            Institution_Address_Line_2__c, Institution_City__c, Institution_State__c,
                            Institution_Postal_Code__c FROM SEED_Project__c
                            WHERE DISTANCE(GeoLocation__c, GEOLOCATION(:latitude, :longitude), 'mi') > 1150
          ORDER BY DISTANCE(GeoLocation__c, GEOLOCATION(:latitude, :longitude), 'mi')]);
        Map<Id, SiteDetailsWrapper> mapSiteIdVsWarpper = new Map<Id, SiteDetailsWrapper>();
        if(!mapProjIdVsDetails.isEmpty()){
            String projectLocation = '';
            for(SEED_Project__c project : mapProjIdVsDetails.values()) {
                SiteDetailsWrapper wrapper = new SiteDetailsWrapper();
                if(mapSiteIdVsWarpper.containsKey(project.Sites__c)){
                    wrapper = mapSiteIdVsWarpper.get(project.Sites__c);
                }
                wrapper.siteId = project.Sites__c;
                wrapper.siteName = project.Sites__r.Name;
                projectLocation = project.Institution_Address_Line_1__c + ', ' + project.Institution_Address_Line_2__c + ', ' +
                     project.Institution_City__c + ', ' + project.Institution_Postal_Code__c + ' ' +
                     project.Institution_State__c;
                wrapper.listOfProjectLocation.add(projectLocation);
                mapSiteIdVsWarpper.put(project.Sites__c, wrapper);
            }
            mapSiteIdVsWarpper = getCoordinatorDetails(mapSiteIdVsWarpper);
        }
        return mapSiteIdVsWarpper;
    }
   
    private static Map<Id, SiteDetailsWrapper> getCoordinatorDetails(Map<Id, SiteDetailsWrapper> mapSiteIdVsWarpper) {
        for(Contact con : [SELECT Name, Email, Phone, Sites__c FROM Contact WHERE Sites__c IN :mapSiteIdVsWarpper.keySet()
                           AND SEED_Contact_Type__c = 'Coordinator']){
            SiteDetailsWrapper wrapper = mapSiteIdVsWarpper.get(con.Sites__c);
            wrapper.coordinatorName = con.Name;
            wrapper.coordinatorEmail = con.Email;
            wrapper.coordinatorEmailTo = 'mailto:'+con.Email;
            wrapper.coordinatorPhone = con.Phone;
            mapSiteIdVsWarpper.put(con.Sites__c, wrapper);                  
        }
        return mapSiteIdVsWarpper;
    }
   
    public class StudentApplicationWrapper {
@AuraEnabled
        public Contact contactDetails;
        @AuraEnabled
        public List<Seed_Student_Application_Step__mdt> listOfSteps;
        @AuraEnabled
        public SeedCoordinatorTimeline__mdt essayQuestions;
        @AuraEnabled
        public SiteDetailsWrapper siteDetails;
        @AuraEnabled
        public List<SiteDetailsWrapper> listSiteDetails;
        public StudentApplicationWrapper(){
            this.listOfSteps = new List<Seed_Student_Application_Step__mdt>();
            this.siteDetails = new SiteDetailsWrapper();
            this.essayQuestions = new SeedCoordinatorTimeline__mdt();
            this.listSiteDetails = new List<SiteDetailsWrapper>();
        }
    }
       
    public class SiteDetailsWrapper {
        @AuraEnabled
        public String siteId;
        @AuraEnabled
        public String siteName;
        @AuraEnabled
        public String coordinatorName;
        @AuraEnabled
        public String coordinatorEmail;
        @AuraEnabled
        public String coordinatorEmailTo;
        @AuraEnabled
        public String coordinatorPhone;
        @AuraEnabled
        public List<String> listOfProjectLocation;
       
        public SiteDetailsWrapper(){
            this.listOfProjectLocation = new List<String>();
        }
    }
   
    public class FetchDetails{
        public Boolean fetchEssayQuestions;
        public Boolean fetchListOfSites;
        public Boolean fetchSiteDetails;
        public FetchDetails(){
            this.fetchEssayQuestions = false;
            this.fetchListOfSites = false;
            this.fetchSiteDetails = false;
        }
    }
}
