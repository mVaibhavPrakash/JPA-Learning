** The best way to use entity inheritance with JPA and Hibernate **

Imagine having a tool that can automatically detect JPA and Hibernate performance issues. Wouldn’t that be just awesome?
Well, Hypersistence Optimizer is that tool! And it works with Spring Boot, Spring Framework, Jakarta EE, Java EE, Quarkus, or Play Framework.
So, enjoy spending your time on the things you love rather than fixing performance issues in your production system on a Saturday night!

=> Introduction

Just like in any OOP (Object-Oriented Programming) language, entity inheritance is suitable for varying behavior rather than reusing data structures, 
for which we could use composition. The Domain Model compromising both data (e.g. persisted entities) and behavior (business logic), 
we can still make use of inheritance for implementing a behavioral software design pattern.

In this article, I’m going to demonstrate how to use JPA inheritance as a means to implement the Strategy design pattern.

Domain Model
Considering we have a notification system that needs to send both email and SMS to customers, we can model the notification relationships as follows:

Both the SmsNotification and EmailNotification inherit the base class Notification properties. However, if we use a RDBMS (relational database system), 
there is no standard way of implementing table inheritance, so we need to emulate this relationship. Usually, there are only two choices:

Either we are using a single table, but then we need to make sure all NOT NULL constraints are enforced via a CHECK of TRIGGER
or we can use separate tables for the base class and subclass entities in which case the subclass table Primary Key is also a Foreign Key to the base class Primary Key.
For this example, we are going to use the JOINED table approach which has the following database entity relationship diagram:

Bridging the gap
With JPA and Hibernate, mapping the OOP and the RDBMS models is straightforward.

The Notification base class is mapped as follows:

@Entity
@Table(name = "notification")
@Inheritance(
    strategy = InheritanceType.JOINED
)
public class Notification {
 
    @Id
    @GeneratedValue
    private Long id;
 
    @Column(name = "first_name")
    private String firstName;
 
    @Column(name = "last_name")
    private String lastName;
 
    @Temporal( TemporalType.TIMESTAMP )
    @CreationTimestamp
    @Column(name = "created_on")
    private Date createdOn;
 
    //Getters and setters omitted for brevity
}
The SmsNotification and EmailNotification mappings looks like this:

@Entity
@Table(name = "sms_notification")
public class SmsNotification
    extends Notification {
 
    @Column(
        name = "phone_number",
        nullable = false
    )
    private String phoneNumber;
 
    //Getters and setters omitted for brevity
}

@Entity
@Table(name = "email_notification")
public class EmailNotification
    extends Notification {
 
    @Column(
        name = "email_address",
        nullable = false
    )
    private String emailAddress;
 
    //Getters and setters omitted for brevity
}
Business logic
So far, we only mapped the relationship between the OOP and the RDBMS data structures, but we haven’t covered the actual business logic which is required to send these notifications to our users.

For this purpose, we have the following NotificationSender Service components:

NotificationSender class diagram

The NotificationSender has two methods:

appliesTo gives the entity that’s supported by this NotificationSender
send encapsulates the actual sending logic
The EmailNotificationSender is implemented as follows:

@Component
public class EmailNotificationSender
    implements NotificationSender<EmailNotification> {
 
    protected final Logger LOGGER = LoggerFactory.getLogger(
        getClass()
    );
 
    @Override
    public Class<EmailNotification> appliesTo() {
        return EmailNotification.class;
    }
 
    @Override
    public void send(EmailNotification notification) {
        LOGGER.info(
            "Send Email to {} {} via address: {}",
             notification.getFirstName(),
             notification.getLastName(),
             notification.getEmailAddress()
        );
    }
}
Of course, the actual sending logic was stripped away, but this is sufficient to understand how the Strategy pattern works.

However, the user does not have to interact with the NotificationSender directly. They only want to send a campaign, and the system should figure out the subscriber channels each client has opted for.

Therefore, we can use the Facade Pattern to expose a very simple API:

NotificationService class diagram

The NotificationSenderImpl is where all the magic happens:

@Service
public class NotificationServiceImpl
    implements NotificationService {
 
    @Autowired
    private NotificationDAO notificationDAO;
 
    @Autowired
    private List<NotificationSender> notificationSenders;
 
    private Map<Class<? extends Notification>, NotificationSender>
        notificationSenderMap = new HashMap<>();
 
    @PostConstruct
    @SuppressWarnings( "unchecked" )
    public void init() {
        for ( NotificationSender notificationSender : notificationSenders ) {
            notificationSenderMap.put(
                notificationSender.appliesTo(),
                notificationSender
            );
        }
    }
 
    @Override
    @Transactional
    @SuppressWarnings( "unchecked" )
    public void sendCampaign(String name, String message) {
        List<Notification> notifications = notificationDAO.findAll();
 
        for ( Notification notification : notifications ) {
            notificationSenderMap
                .get( notification.getClass() )
                .send( notification );
        }
    }
}
There are several things to note in this implementation:

We make use of Spring List auto-wiring feature which I explained in my very first blog post. This way, we can inject any NotificationSender the user has configured in our system, therefore decoupling the NotificationService from the actual NotificationSender implementations our system us currently supporting.
The init method builds the notificationSenderMap which takes a Notification class type as the Map key and the associated NotificationSender as the Map value.
The sendCampaign method fetches a List of Notification entities from the DAO layer and pushes them to their associated NotificationSender instances.
Because JPA offers polymorphic queries, the findAll DAO method can be implemented as follows:

@Override
public List<T> findAll() {
    CriteriaBuilder builder = entityManager
        .getCriteriaBuilder();
         
    CriteriaQuery<T> criteria = builder
        .createQuery( entityClass );
    criteria.from( entityClass );
 
    return entityManager
        .createQuery( criteria )
        .getResultList();
}
Writing JPA Criteria API queries is not very easy. The Codota IDE plugin can guide you on how to write such queries, therefore increasing your productivity.

For more details about how you can use Codota to speed up the process of writing Criteria API queries, check out this article.

The system does not have to know which are the actual Notification implementation each client has chosen. The polymorphic query is figured out at runtime by JPA and Hibernate.

Testing time
If we created the following Notification entities in our system:

SmsNotification sms = new SmsNotification();
sms.setPhoneNumber( "012-345-67890" );
sms.setFirstName( "Vlad" );
sms.setLastName( "Mihalcea" );
 
entityManager.persist( sms );
 
EmailNotification email = new EmailNotification();
email.setEmailAddress( "vlad@acme.com" );
email.setFirstName( "Vlad" );
email.setLastName( "Mihalcea" );
 
entityManager.persist( email );
And now we want to send a campaign:

notificationService.sendCampaign(
    "Black Friday",
    "High-Performance Java Persistence is 40% OFF"
);
Hibernate executes the following SQL query:

SELECT 
    n.id AS id1_1_,
    n.created_on AS created_2_1_,
    n.first_name AS first_na3_1_,
    n.last_name AS last_nam4_1_,
    n1_.email_address AS email_ad1_0_,
    n2_.phone_number AS phone_nu1_2_,
    CASE WHEN n1_.id IS NOT NULL THEN 1
         WHEN n2_.id IS NOT NULL THEN 2
         WHEN n.id IS NOT NULL THEN 0
    END AS clazz_
FROM   
    notification n
LEFT OUTER JOIN
    email_notification n1_ ON n.id = n1_.id
LEFT OUTER JOIN
    sms_notification n2_ ON n.id = n2_.id

And the following output is logged:

EmailNotificationSender - Send Email to Vlad Mihalcea via address: vlad@acme.com
 
SmsNotificationSender - Send SMS to Vlad Mihalcea via phone number: 012-345-67890

** Composition with embeddables **

=> Embeddables are another option to use composition when implementing your entities. They enable you to define a reusable set of attributes with mapping annotations. 
In contrast to the previously discussed association mappings, the embeddable becomes part of the entity and has no persistent identity on its own.

Let’s take a look at an example.

The 3 attributes of the Address class store simple address information. The @Embeddable annotation tells Hibernate 
and any other JPA implementation that this class and its mapping annotations can be embedded into an entity. 
In this example, I rely on JPA’s default mappings and don’t provide any mapping information.

@Embeddable
public class Address {
 
    private String street;
    private String city;
    private String postalCode;
     
    ...
}
After you have defined your embeddable, you can use it as the type of an entity attribute. 
You just need to annotate it with @Embedded, and your persistence provider will include the attributes and mapping information of your embeddable in the entity.

So, in this example, the attributes street, city and postalCode of the embeddable Address will be mapped to columns of the Author table.

@Entity
public class Author implements Serializable {
 
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
     
    @Embedded
    private Address address;
     
    ...
}
If you want to use multiple attributes of the same embeddable type, you need to override the column mappings of the attributes of the embeddable.
You can do that with a collection of @AttributeOverride annotations. Since JPA 2.2, the @AttributeOverride annotation is repeatable, 
and you no longer need to wrap it in an @AttributeOverrides annotation.

@Entity
public class Author implements Serializable {
 
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
     
    @Embedded
    private Address privateAddress;
     
    @Embedded
    @AttributeOverride(
        name = "street",
        column = @Column( name = "business_street" )
    )
    @AttributeOverride(
        name = "city",
        column = @Column( name = "business_city" )
    )
    @AttributeOverride(
        name = "postalCode",
        column = @Column( name = "business_postcalcode" )
    )
    private Address businessAddress;
     
    ...
}