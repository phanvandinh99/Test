# Test

KAFKA
student-management/
│── src/
│   ├── main/
│   │   ├── java/com/example/studentmanagement/
│   │   │   ├── config/
│   │   │   │   ├── KafkaConfig.java
│   │   │   ├── controller/
│   │   │   │   ├── NotificationController.java
│   │   │   ├── entity/
│   │   │   │   ├── Student.java
│   │   │   │   ├── Classroom.java
│   │   │   │   ├── Notification.java
│   │   │   ├── repository/
│   │   │   │   ├── StudentRepository.java
│   │   │   │   ├── ClassroomRepository.java
│   │   │   │   ├── NotificationRepository.java
│   │   │   ├── service/
│   │   │   │   ├── NotificationService.java
│   │   │   ├── kafka/
│   │   │   │   ├── NotificationProducer.java
│   │   │   │   ├── NotificationConsumer.java
│   │   │   ├── StudentManagementApplication.java
│   ├── resources/
│   │   ├── application.yml
│   ├── test/
│── pom.xml
│── docker-compose.yml
│── README.md

docker-compose up –d

version: '3'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

docker-compose up –d

Cấu hình Kafka trong  application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    consumer:
      group-id: "student-group"
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer

Consumer Kafka - Nhận thông báo

@Component
@Slf4j
public class NotificationConsumer {

    @KafkaListener(topics = "notification_topic", groupId = "student-group")
    public void consume(String message) {
        log.info("Received Notification: {}", message);
    }
}


@RestController
@RequestMapping("/notifications")
@RequiredArgsConstructor
public class NotificationController {
    private final NotificationProducer producer;
    private final StudentRepository studentRepository;
    private final NotificationRepository notificationRepository;
    private final ClassroomRepository classroomRepository;

    @PostMapping("/send/{classroomId}")
    public ResponseEntity<String> sendNotification(@PathVariable Long classroomId, @RequestBody String message) {
        Classroom classroom = classroomRepository.findById(classroomId)
                .orElseThrow(() -> new RuntimeException("Classroom not found"));

        Notification notification = new Notification();
        notification.setMessage(message);
        notification.setClassroom(classroom);
        notificationRepository.save(notification);

        // Gửi tin nhắn đến tất cả sinh viên trong lớp qua Kafka
        List<Student> students = studentRepository.findByClassroomId(classroomId);
        students.forEach(student -> {
            String msg = "Gửi đến " + student.getName() + ": " + message;
            producer.sendNotification(msg);
        });

        return ResponseEntity.ok("Notification sent to class " + classroom.getName());
    }
}


docker-compose up -d
