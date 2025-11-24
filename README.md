# SmartStudentTaskManager
A Java project for VIT flipped course requirements
Problem Statement:
Students often struggle to manage tasks, deadlines, and study schedules. This tool provides a lightweight desktop CLI application to create, prioritize and track study tasks and deadlines, helping students focus and plan.

Scope:
- Student users (single-user per profile)
- Task management (add, edit, remove, mark complete)
- Prioritization & suggestion engine
- Reports and persistence

Target users:
- Undergraduate students using the flipped course
- Anyone who prefers a lightweight Java desktop/task CLI utility

High-level features:
- User management
- Task CRUD
- Smart suggestion engine
- Persistence and reporting



JAVA CODES 
// File: src/main/java/com/ssstm/model/Task.java
package com.ssstm.model;

import java.io.Serializable;
import java.time.LocalDateTime;
import java.util.UUID;
import java.util.List;
import java.util.ArrayList;

/**
 * Represents a task.
 */
public class Task implements Serializable {
    private static final long serialVersionUID = 1L;

    public enum Priority { LOW, MEDIUM, HIGH }

    private final String id;
    private String title;
    private String description;
    private LocalDateTime dueDate;
    private Priority priority;
    private int estimatedMinutes;
    private boolean completed;
    private List<String> tags;

    public Task(String title, String description, LocalDateTime dueDate,
                Priority priority, int estimatedMinutes, List<String> tags) {
        this.id = UUID.randomUUID().toString();
        this.title = title;
        this.description = description;
        this.dueDate = dueDate;
        this.priority = priority;
        this.estimatedMinutes = estimatedMinutes;
        this.completed = false;
        this.tags = tags != null ? new ArrayList<>(tags) : new ArrayList<>();
    }

    // Getters and setters
    public String getId() { return id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public LocalDateTime getDueDate() { return dueDate; }
    public void setDueDate(LocalDateTime dueDate) { this.dueDate = dueDate; }
    public Priority getPriority() { return priority; }
    public void setPriority(Priority priority) { this.priority = priority; }
    public int getEstimatedMinutes() { return estimatedMinutes; }
    public void setEstimatedMinutes(int estimatedMinutes) { this.estimatedMinutes = estimatedMinutes; }
    public boolean isCompleted() { return completed; }
    public void setCompleted(boolean completed) { this.completed = completed; }
    public List<String> getTags() { return tags; }
    public void setTags(List<String> tags) { this.tags = tags; }

    @Override
    public String toString() {
        return String.format("[%s] %s (Due: %s) Priority: %s EstMin: %d Completed: %s",
                id.substring(0,8), title, dueDate == null ? "N/A" : dueDate.toString(),
                priority, estimatedMinutes, completed ? "YES" : "NO");
    }
}

// File: src/main/java/com/ssstm/model/User.java
package com.ssstm.model;

import java.io.Serializable;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

public class User implements Serializable {
    private static final long serialVersionUID = 1L;

  private String username;
    private LocalDateTime createdAt;
    private List<Task> tasks;

  public User(String username) {
        this.username = username;
        this.createdAt = LocalDateTime.now();
        this.tasks = new ArrayList<>();
    }

   public String getUsername() { return username; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public List<Task> getTasks() { return tasks; }

  public void addTask(Task t) { tasks.add(t); }
    public void removeTask(Task t) { tasks.remove(t); }
}

// File: src/main/java/com/ssstm/storage/Storage.java
package com.ssstm.storage;

import com.ssstm.model.User;

import java.io.*;
import java.nio.file.*;
import java.util.HashMap;
import java.util.Map;

/**
 * Simple file-based storage using Java serialization.
 */
public class Storage {
    private static final String DATA_DIR = "data";
    private static final String USERS_FILE = DATA_DIR + "/users.ser";
    private static final String USERS_BACKUP = DATA_DIR + "/users.ser.bak";

    // Save users map to file
    public static void saveUsers(Map<String, User> users) throws IOException {
        Files.createDirectories(Paths.get(DATA_DIR));
        // backup previous file
        if (Files.exists(Paths.get(USERS_FILE))) {
            Files.copy(Paths.get(USERS_FILE), Paths.get(USERS_BACKUP), StandardCopyOption.REPLACE_EXISTING);
        }

   try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(USERS_FILE))) {
            oos.writeObject(users);
        }
    }

    // Load users map from file, return empty map if not present or on error
    @SuppressWarnings("unchecked")
    public static Map<String, User> loadUsers() {
        try {
            File f = new File(USERS_FILE);
            if (!f.exists()) return new HashMap<>();
            try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(f))) {
                Object o = ois.readObject();
                if (o instanceof Map) {
                    return (Map<String, User>) o;
                } else {
                    System.err.println("Data format mismatch.");
                }
            }
        } catch (Exception ex) {
            System.err.println("Failed to load users: " + ex.getMessage());
        }
        return new HashMap<>();
    }
}

// File: src/main/java/com/ssstm/service/UserService.java
package com.ssstm.service;

import com.ssstm.model.User;
import com.ssstm.storage.Storage;

import java.io.IOException;
import java.util.Map;
import java.util.HashMap;

/**
 * Manages user creation and retrieval.
 */
public class UserService {
    private Map<String, User> users;

    public UserService() {
        users = Storage.loadUsers();
    }

    public boolean usernameExists(String username) {
        return users.containsKey(username.toLowerCase());
    }

    public User createUser(String username) {
        String key = username.toLowerCase();
        if (users.containsKey(key)) return null;
        User u = new User(username);
        users.put(key, u);
        persist();
        return u;
    }

    public User getUser(String username) {
        if (username == null) return null;
        return users.get(username.toLowerCase());
    }

    public void persist() {
        try {
            Storage.saveUsers(users);
        } catch (IOException e) {
            System.err.println("Failed to save data: " + e.getMessage());
        }
    }

    public Map<String, User> getAllUsers() { return users; }
}

// File: src/main/java/com/ssstm/service/TaskManager.java
package com.ssstm.service;

import com.ssstm.model.Task;
import com.ssstm.model.User;

import java.time.Duration;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Core task operations and suggestion algorithm.
 */
public class TaskManager {
    private final UserService userService;

    public TaskManager(UserService userService) {
        this.userService = userService;
    }

    public Task addTask(User user, Task task) {
        user.addTask(task);
        userService.persist();
        return task;
    }

    public boolean deleteTask(User user, String taskId) {
        Optional<Task> t = findById(user, taskId);
        if (t.isPresent()) {
            user.removeTask(t.get());
            userService.persist();
            return true;
        }
        return false;
    }

    public Optional<Task> findById(User user, String id) {
        return user.getTasks().stream().filter(x -> x.getId().equals(id)).findFirst();
    }

    public List<Task> listTasks(User user) {
        return new ArrayList<>(user.getTasks());
    }

    // Simple recommendation: compute score and return highest non-completed
    public Optional<Task> recommendNext(User user) {
        LocalDateTime now = LocalDateTime.now();
        return user.getTasks().stream().filter(t -> !t.isCompleted())
            .max(Comparator.comparingDouble(t -> computeScore(t, now)));
    }

    // Score heuristic: nearer dueDate -> higher score; high priority -> +; shorter estimated time slightly favored
    private double computeScore(Task t, LocalDateTime now) {
        double score = 0;
        long hoursLeft = (t.getDueDate() == null) ? 99999 : Duration.between(now, t.getDueDate()).toHours();
        // due weight
        if (hoursLeft <= 0) score += 1000; // overdue highest
        else score += Math.max(0, 100 - Math.min(hoursLeft, 100));

    // priority weight
        switch (t.getPriority()) {
            case HIGH: score += 200; break;
            case MEDIUM: score += 100; break;
            default: score += 10;
        }

   // estimated time (prefer shorter tasks if same urgency)
        score += Math.max(0, 60 - t.getEstimatedMinutes()) * 0.5;

    // completed? (shouldn't be recommended)
        if (t.isCompleted()) score -= 10000;
        return score;
    }

    // Reports
    public List<Task> tasksDueWithin(User user, Duration duration) {
        LocalDateTime now = LocalDateTime.now();
        LocalDateTime until = now.plus(duration);
        return user.getTasks().stream()
                .filter(t -> !t.isCompleted() && t.getDueDate() != null && !t.getDueDate().isBefore(now) && !t.getDueDate().isAfter(until))
                .collect(Collectors.toList());
    }

    public List<Task> overdueTasks(User user) {
        LocalDateTime now = LocalDateTime.now();
        return user.getTasks().stream()
                .filter(t -> !t.isCompleted() && t.getDueDate()!=null && t.getDueDate().isBefore(now))
                .collect(Collectors.toList());
    }

    // Expose persist via TaskManager for convenience
    public void persist() {
        userService.persist();
    }
}

// File: src/main/java/com/ssstm/service/ReminderService.java
package com.ssstm.service;

import com.ssstm.model.Task;
import com.ssstm.model.User;

import java.time.Duration;
import java.util.List;

/**
 * Reminder helpers.
 */
public class ReminderService {
    private final TaskManager taskManager;

    public ReminderService(TaskManager taskManager) {
        this.taskManager = taskManager;
    }

    public List<Task> tasksDueSoon(User user) {
        // tasks due within next 24 hours
        return taskManager.tasksDueWithin(user, Duration.ofHours(24));
    }

    public List<Task> overdue(User user) {
        return taskManager.overdueTasks(user);
    }
}

// File: src/main/java/com/ssstm/app/App.java
package com.ssstm.app;

import com.ssstm.model.Task;
import com.ssstm.model.Task.Priority;
import com.ssstm.model.User;
import com.ssstm.service.TaskManager;
import com.ssstm.service.UserService;
import com.ssstm.service.ReminderService;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;

/**
 * Main CLI application.
 */
public class App {
    private static final Scanner scanner = new Scanner(System.in);
    private static final DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

    public static void main(String[] args) {
        UserService userService = new UserService();
        TaskManager taskManager = new TaskManager(userService);
        ReminderService reminderService = new ReminderService(taskManager);

    System.out.println("=== Smart Student Task & Time Manager ===");
        User current = selectOrCreateUser(userService);

    // Show reminders
        List<Task> dueSoon = reminderService.tasksDueSoon(current);
        if (!dueSoon.isEmpty()) {
            System.out.println("\n!! Tasks due within 24 hours !!");
            dueSoon.forEach(t -> System.out.println(" - " + t));
        }
        List<Task> overdue = reminderService.overdue(current);
        if (!overdue.isEmpty()) {
            System.out.println("\n!! Overdue tasks !!");
            overdue.forEach(t -> System.out.println(" - " + t));
        }

     boolean running = true;
        while (running) {
            printMenu();
            String choice = scanner.nextLine().trim();
            switch (choice) {
                case "1": addTaskFlow(current, taskManager); break;
                case "2": listTasksFlow(current, taskManager); break;
                case "3": markCompleteFlow(current, taskManager); break;
                case "4": recommendFlow(current, taskManager); break;
                case "5": deleteFlow(current, taskManager); break;
                case "6": taskManager.persist(); System.out.println("Saved."); break;
                case "0":
                    taskManager.persist();
                    System.out.println("Goodbye!"); running = false; break;
                default: System.out.println("Invalid choice.");
            }
        }
    }

    private static User selectOrCreateUser(UserService us) {
        System.out.println("Existing users: " + us.getAllUsers().keySet());
        System.out.print("Enter username to select or new username to create: ");
        String name = scanner.nextLine().trim();
        User u = us.getUser(name);
        if (u == null) {
            u = us.createUser(name);
            System.out.println("Created user: " + name);
        } else {
            System.out.println("Loaded user: " + name);
        }
        return u;
    }

    private static void printMenu() {
        System.out.println("\nMenu:");
        System.out.println("1) Add Task");
        System.out.println("2) List Tasks");
        System.out.println("3) Mark Task Complete");
        System.out.println("4) Recommend Next Task");
        System.out.println("5) Delete Task");
        System.out.println("6) Save Now");
        System.out.println("0) Exit");
        System.out.print("Choice: ");
    }

    private static void addTaskFlow(User user, TaskManager tm) {
        try {
            System.out.print("Title: "); String title = scanner.nextLine();
            System.out.print("Description: "); String desc = scanner.nextLine();
            System.out.print("Due date (yyyy-MM-dd HH:mm) or leave empty: "); String d = scanner.nextLine();
            LocalDateTime due = null;
            if (!d.isBlank()) due = LocalDateTime.parse(d, dtf);
            System.out.print("Priority (LOW/MEDIUM/HIGH): "); Priority p = Priority.valueOf(scanner.nextLine().trim().toUpperCase());
            System.out.print("Estimated minutes: "); int est = Integer.parseInt(scanner.nextLine().trim());
            System.out.print("Tags (comma separated): "); String tagsLine = scanner.nextLine().trim();
            List<String> tags = tagsLine.isEmpty() ? new ArrayList<>() : Arrays.asList(tagsLine.split("\\s*,\\s*"));

    Task t = new Task(title, desc, due, p, est, tags);
            tm.addTask(user, t);
            System.out.println("Added: " + t);
        } catch (Exception ex) {
            System.out.println("Failed to add task: " + ex.getMessage());
        }
    }

    private static void listTasksFlow(User user, TaskManager tm) {
        List<Task> tasks = tm.listTasks(user);
        if (tasks.isEmpty()) {
            System.out.println("No tasks.");
            return;
        }
        System.out.println("Tasks:");
        tasks.forEach(t -> System.out.println(" - " + t));
    }

    private static void markCompleteFlow(User user, TaskManager tm) {
        System.out.print("Enter task id (prefix allowed): ");
        String id = scanner.nextLine().trim();
        Optional<Task> found = user.getTasks().stream().filter(t -> t.getId().startsWith(id)).findFirst();
        if (found.isPresent()) {
            found.get().setCompleted(true);
            tm.persist();
            System.out.println("Marked complete: " + found.get());
        } else {
            System.out.println("Task not found.");
        }
    }

    private static void recommendFlow(User user, TaskManager tm) {
        Optional<Task> t = tm.recommendNext(user);
        System.out.println(t.map(task -> "Recommended: " + task).orElse("No recommendation (all done?)"));
    }

    private static void deleteFlow(User user, TaskManager tm) {
        System.out.print("Enter full or starting task id to delete: ");
        String id = scanner.nextLine().trim();
        Optional<Task> found = user.getTasks().stream().filter(t -> t.getId().startsWith(id)).findFirst();
        if (found.isPresent()) {
            tm.deleteTask(user, found.get().getId());
            System.out.println("Deleted.");
        } else {
            System.out.println("Task not found.");
        }
    }
}
