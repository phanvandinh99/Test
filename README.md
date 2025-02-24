lydinc-backend/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── lydinc-backend/                               # Package chính của ứng dụng
│   │   │           ├── config/                                   # Chứa các file cấu hình hệ thống
│   │   │           │   ├── KafkaConfig.java                      # Cấu hình Kafka (Producer, Consumer)
│   │   │           │   ├── WebSocketConfig.java                  # Cấu hình WebSocket để gửi thông báo real-time
│   │   │           ├── controller/                               # Chứa các API Controller
│   │   │           │   ├── NotificationController.java           # API xử lý thông báo (gửi, nhận)
│   │   │           ├── entity/                                   # Chứa các Entity ánh xạ với database
│   │   │           │   ├── Example.java                          # Chứa các Entity
│   │   │           ├── repository/                               # Chứa các repository thao tác với database
│   │   │           │   ├── NotificationRepository.java           # Repository thao tác với bảng Notification
│   │   │           │   ├── UserRepository.java                   # Repository thao tác với bảng User
│   │   │           │   ├── CourseRepository.java                 # Repository thao tác với bảng Course
│   │   │           ├── service/                                  # Chứa các service xử lý logic nghiệp vụ
│   │   │           │   ├── NotificationService.java              # Service xử lý logic gửi & nhận thông báo
│   │   │           │   ├── KafkaProducerService.java             # Service gửi thông báo qua Kafka (Producer)
│   │   │           │   ├── KafkaConsumerService.java             # Service nhận thông báo từ Kafka (Consumer)
│   │   │           ├── websocket/                                # Chứa xử lý WebSocket
│   │   │           │   ├── NotificationWebSocketHandler.java     # Xử lý WebSocket push thông báo real-time
│   │   │           └── LydincBackendApplication.java             # Main class chạy ứng dụng Spring Boot
│   │   └── resources/                                            # Chứa các file cấu hình và tài nguyên
│   │       ├── application.yml                                   # File cấu hình ứng dụng (database, Kafka, WebSocket)
│   └── test/                                                     # Chứa các file test ứng dụng
├── pom.xml                                                       # File cấu hình Maven (dependencies, build)
└── docker-compose.yaml                                           # Cấu hình Docker Compose (Kafka, PostgreSQL)


## api
# 2.1. Dependencies (pom.xml)
<!-- Kafka -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
<!-- WebSocket -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>

# 2.2. Configuration
<!-- pplication.yml -->
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: notification-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring:
          json:
            trusted:
              packages: '*'
server:
  port: 8080


<!-- KafkaConfig.java -->
package com.lydinc.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaConfig {
    @Bean
    public NewTopic notificationTopic() {
        return TopicBuilder.name("notification-topic")
                .partitions(1)
                .replicas(1)
                .build();
    }
}


<!-- WebSocketConfig.java -->
package com.lydinc.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;
import com.lydinc.websocket.NotificationWebSocketHandler;
import lombok.RequiredArgsConstructor;

@Configuration
@EnableWebSocket
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketConfigurer {
    private final NotificationWebSocketHandler notificationWebSocketHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(notificationWebSocketHandler, "/notifications").setAllowedOrigins("*");
    }
}

## 2.4. Kafka Producer
<!-- KafkaProducerService.java -->
package com.lydinc.service;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import com.lydinc.entity.Notification;

@Service
public class KafkaProducerService {
    private final KafkaTemplate<String, Notification> kafkaTemplate;

    public KafkaProducerService(KafkaTemplate<String, Notification> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendNotification(Notification notification) {
        kafkaTemplate.send("notification-topic", notification);
    }
}

## 2.5. Kafka Consumer
<!-- KafkaConsumerService.java -->
package com.lydinc.service;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import com.lydinc.entity.Notification;
import com.lydinc.websocket.NotificationWebSocketHandler;
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class KafkaConsumerService {
    private final NotificationWebSocketHandler webSocketHandler;

    public KafkaConsumerService(NotificationWebSocketHandler webSocketHandler) {
        this.webSocketHandler = webSocketHandler;
    }

    @KafkaListener(topics = "notification-topic", groupId = "notification-group")
    public void listen(Notification notification) {
        webSocketHandler.sendNotificationToClients(notification);
    }
}

## 2.6. WebSocket Handler
<!-- NotificationWebSocketHandler.java -->
package com.lydinc.websocket;

import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.lydinc.entity.Notification;
import java.util.ArrayList;
import java.util.List;

@Component
public class NotificationWebSocketHandler extends TextWebSocketHandler {
    private final List<WebSocketSession> sessions = new CopyOnWriteArrayList<>();
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        sessions.add(session);
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        sessions.remove(session);
    }

    public void sendNotificationToClients(Notification notification) {
        try {
            String message = objectMapper.writeValueAsString(notification);
            for (WebSocketSession session : sessions) {
                if (session.isOpen()) {
                    session.sendMessage(new TextMessage(message));
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

## 2.7. Service
<!-- NotificationService.java -->
package com.lydinc.service;

import org.springframework.stereotype.Service;
import com.lydinc.entity.Course;
import com.lydinc.entity.Notification;
import com.lydinc.entity.User;
import com.lydinc.repository.NotificationRepository;
import com.lydinc.repository.UserRepository;

import lombok.RequiredArgsConstructor;

import java.time.LocalDateTime;
import java.util.List;

@Service
@RequiredArgsConstructor
public class NotificationService {
    private final NotificationRepository notificationRepository;
    private final UserRepository userRepository;
    private final KafkaProducerService kafkaProducerService;
    private final CourseRepository courseRepository;

    public void sendNotificationToCourseStudents(Long courseId, String message) {
        Course course = courseRepository.findById(courseId)
                .orElseThrow(() -> new RuntimeException("Course not found"));
        List<User> students = userRepository.findByCourses(course);
        for (User student : students) {
            Notification notification = new Notification();
            notification.setMessage(message);
            notification.setCourse(course);
            notification.setUser(student);
            notification.setCreatedAt(LocalDateTime.now());
            notificationRepository.save(notification);
            kafkaProducerService.sendNotification(notification);
        }
    }
}

## 2.8. Controller
<!-- NotificationController.java -->
package com.lydinc.controller;

import org.springframework.web.bind.annotation.*;
import com.lydinc.entity.Course;
import com.lydinc.service.NotificationService;

import lombok.RequiredArgsConstructor;

@RestController
@RequestMapping("/api/notifications")
@RequiredArgsConstructor
public class NotificationController {
    private final NotificationService notificationService;
    private final CourseRepository courseRepository;

    @PostMapping("/send")
    public ResponseEntity<Void> sendNotification(@RequestParam Long courseId, @RequestBody String message) {
        Course course = courseRepository.findById(courseId)
                .orElseThrow(() -> new RuntimeException("Course not found"));
        notificationService.sendNotificationToCourseStudents(course.getId(), message);
        return ResponseEntity.ok().build();
    }
}

## 3. Docker Compose (Kafka)
<!-- docker-compose.yaml -->
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

<!-- Run docker composer-->
docker-compose up -d


## 4. Frontend: ReactJS
## 4.1. Cài đặt WebSocket
npm install socket.io-client

## 4.2. Notification Component
<!-- src/components/Notification.js -->
import React, { useEffect, useState } from 'react';

const Notification = () => {
    const [notifications, setNotifications] = useState([]);

    useEffect(() => {
        const socket = new WebSocket('ws://localhost:8080/notifications');

        socket.onopen = () => {
            console.log('WebSocket connected');
        };

        socket.onmessage = (event) => {
            const notification = JSON.parse(event.data);
            setNotifications((prev) => [...prev, notification]);
        };

        socket.onerror = (error) => {
            console.error('WebSocket error:', error);
        };

        socket.onclose = () => {
            console.log('WebSocket disconnected');
        };

        return () => {
            socket.close();
        };
    }, []);

    return (
        <div>
            <h2>Notifications</h2>
            <ul>
                {notifications.map((notif, index) => (
                    <li key={index}>
                        {notif.message} - {new Date(notif.createdAt).toLocaleString()}
                    </li>
                ))}
            </ul>
        </div>
    );
};

export default Notification;


## 4.3. App.js
import React from 'react';
import Notification from './components/Notification';

function App() {
    return (
        <div>
            <h1>Lydinc Frontend</h1>
            <Notification />
        </div>
    );
}

export default App;

## 5. Cách chạy
Backend:
  Build: mvn clean package
  Run: mvn spring-boot:run

rontend:
  Run: npm start