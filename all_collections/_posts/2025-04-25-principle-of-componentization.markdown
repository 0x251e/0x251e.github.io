---
layout: post
title: Principle of Componentization
date: 2025-04-25
categories: [software engineering]
---

### Prologue
As I dive into Chapter 10 of Effective C by Robert C. Seacord, one key concept stands out: the **Principle of Componentization**. It is about more than just splitting code into separate files—it is about designing systems that are modular, maintainable and scalable. This principle touches on vital software design concepts like coupling and cohesion, code reuse, data abstraction and opaque types. 

While I am still exploring practical examples for each of these areas, this post captures my current understanding and thoughts as I connect these ideas to how we write better C programs. Think of it as my midpoint reflection with technical in nature, but still open to my growth as my journey continues.

### Principles of Componentization
Componentization is the process of breaking down a complex codebase into a smaller, reusable parts or components which will be manageable easily and understandable. Moreover, the broken down components can be reuse elsewhere in the program or even in other program. 

The Principle of Componentization is crucial because it brings order to complexity. As C programs grow in size and scope, maintaining a monolithic structure quickly becomes unmanagable. Componentization allows developers to isolate functionality, reduce interdependencies, and promote reusability. This not only improves code readability and debugging but also enables teams to work in parallel and update individual components without breaking the entire system. In short, it lays the foundation of scalable, maintainable and reliable software. 

### Coupling and Cohension
A well-structured program will have the properties of **low coupling and high cohesion**. 
- **Cohesion**: A measure of the commonality between elements of programming interface. For example, a header file that groups related string manipulation functions (like `strlen`,`strcat`,`strstr`) demonstrates high cohension. This makes it intuitive to use and easier to maintain, as all related functionality resides in one place. 
- **Coupling**: A measure of the interdependency of programming interfaces. Tightly coupled components rely on each other heavily—sometimes even requiring specific inclusion order—making it fragile and harder to modify. Loose coupling on other hand, ensures components can be understood, tested, and changed independently, which promotes flexibility and reduces bugs.

Here is an examples that shows good and bad practices for coupling and cohesion in C:
```c
/* 
 * Component 1: Temperature Sensor Module
 * High cohesion - all functions relate to temperature sensing
 * Low coupling - minimal dependencies on other components
 */

// temperature_sensor.h
#ifndef TEMPERATURE_SENSOR_H
#define TEMPERATURE_SENSOR_H

typedef struct {
    float current_temp;
    float min_temp;
    float max_temp;
} TemperatureData;

void temp_sensor_init(void);
TemperatureData temp_sensor_read(void);
void temp_sensor_calibrate(float offset);

#endif

// temperature_sensor.c
#include "temperature_sensor.h"
#include "hardware_specific.h" // Only hardware-specific dependency

static TemperatureData sensor_data;

void temp_sensor_init(void) {
    sensor_data.current_temp = 0.0f;
    sensor_data.min_temp = -20.0f;
    sensor_data.max_temp = 100.0f;
    hardware_init();
}

TemperatureData temp_sensor_read(void) {
    sensor_data.current_temp = read_hardware_temp();
    return sensor_data;
}

void temp_sensor_calibrate(float offset) {
    sensor_data.current_temp += offset;
}

/*
 * Component 2: Temperature Display Module
 * High cohesion - all about displaying temperature
 * Low coupling - interacts with sensor through clean interface
 */

// temperature_display.h
#ifndef TEMPERATURE_DISPLAY_H
#define TEMPERATURE_DISPLAY_H

void display_init(void);
void display_temperature(float temp);

#endif

// temperature_display.c
#include "temperature_display.h"
#include "temperature_sensor.h"
#include "screen_driver.h" // Only screen-specific dependency

void display_init(void) {
    screen_init();
}

void display_temperature(float temp) {
    char buffer[20];
    sprintf(buffer, "Temp: %.1fC", temp);
    screen_print(buffer);
}

/*
 * Main application - coordinates components
 */
int main() {
    temp_sensor_init();
    display_init();
    
    while(1) {
        TemperatureData data = temp_sensor_read();
        display_temperature(data.current_temp);
        delay(1000);
    }
    
    return 0;
}
```

```c
/*
 * Bad example with low cohesion and high coupling
 * Everything is mixed together with no clear separation
 */

// globals.h (shared global state - promotes high coupling)
float current_temp;
int screen_initialized = 0;
float temp_offset = 0.0f;

// mixed_functions.c (low cohesion - unrelated functions together)
#include "globals.h"
#include "hardware_specific.h"
#include "screen_driver.h"

void initialize_stuff() {
    // Mixes hardware and screen initialization
    hardware_init();
    screen_init();
    screen_initialized = 1;
    current_temp = 0.0f;
}

float get_temp() {
    // Mixes reading and calibration
    float raw = read_hardware_temp();
    return raw + temp_offset;
}

void display_stuff() {
    // Directly depends on screen state
    if (!screen_initialized) return;
    
    float temp = get_temp();
    char buffer[20];
    sprintf(buffer, "Temp: %.1f", temp);
    screen_print(buffer);
}

void set_calibration(float offset) {
    // Modifies global state used by multiple functions
    temp_offset = offset;
}

int main() {
    initialize_stuff();
    
    while(1) {
        display_stuff();
        delay(1000);
    }
    
    return 0;
}
```

### Code Reuse
Code reuse is the practice of writing functionality once and using it multiple places, rather than duplicating code. Duplication not only bloats the codebase and executable size but also increases the risk of inconsistent behavior and higher maintenance overhead. Reusable logic should be encapsulated in functions, especially when the same task—or a slightly varied version—is needed more than once. Parameterization can help adapt functions for multiple scenarios. Leveraging exisiting, well-known functions like `strlen` not only improves readability and reduce bugs but also makes it easier to maintain and update code. However, it is important to strike a balance—interfaces should be general enough for reuse, yet specific enough to be efficient and relevant to current needs. 

Here's an example that shows good and bad practices for code reuse in C, building on the previous coupling/cohesion example:

```c
/*
 * Reusable Component: Circular Buffer
 * Can be used in multiple projects without modification
 */

// circular_buffer.h
#ifndef CIRCULAR_BUFFER_H
#define CIRCULAR_BUFFER_H

#include <stdbool.h>

typedef struct {
    float *buffer;
    int head;
    int tail;
    int size;
    int capacity;
} CircularBuffer;

CircularBuffer* cb_create(int capacity);
void cb_destroy(CircularBuffer *cb);
bool cb_is_empty(const CircularBuffer *cb);
bool cb_is_full(const CircularBuffer *cb);
bool cb_push(CircularBuffer *cb, float value);
bool cb_pop(CircularBuffer *cb, float *value);
int cb_count(const CircularBuffer *cb);

#endif

// circular_buffer.c
#include "circular_buffer.h"
#include <stdlib.h>

CircularBuffer* cb_create(int capacity) {
    CircularBuffer *cb = malloc(sizeof(CircularBuffer));
    if (!cb) return NULL;
    
    cb->buffer = malloc(capacity * sizeof(float));
    if (!cb->buffer) {
        free(cb);
        return NULL;
    }
    
    cb->head = 0;
    cb->tail = 0;
    cb->size = 0;
    cb->capacity = capacity;
    return cb;
}

void cb_destroy(CircularBuffer *cb) {
    if (cb) {
        free(cb->buffer);
        free(cb);
    }
}

bool cb_is_empty(const CircularBuffer *cb) {
    return cb->size == 0;
}

bool cb_is_full(const CircularBuffer *cb) {
    return cb->size == cb->capacity;
}

bool cb_push(CircularBuffer *cb, float value) {
    if (cb_is_full(cb)) return false;
    
    cb->buffer[cb->head] = value;
    cb->head = (cb->head + 1) % cb->capacity;
    cb->size++;
    return true;
}

bool cb_pop(CircularBuffer *cb, float *value) {
    if (cb_is_empty(cb)) return false;
    
    *value = cb->buffer[cb->tail];
    cb->tail = (cb->tail + 1) % cb->capacity;
    cb->size--;
    return true;
}

int cb_count(const CircularBuffer *cb) {
    return cb->size;
}

/*
 * Reusing the circular buffer in temperature monitoring
 */

// temperature_history.h
#ifndef TEMPERATURE_HISTORY_H
#define TEMPERATURE_HISTORY_H

void temp_history_init(int capacity);
void temp_history_record(float temp);
float temp_history_get_average(void);
void temp_history_cleanup(void);

#endif

// temperature_history.c
#include "temperature_history.h"
#include "circular_buffer.h"

static CircularBuffer *temp_buffer;

void temp_history_init(int capacity) {
    temp_buffer = cb_create(capacity);
}

void temp_history_record(float temp) {
    if (!temp_buffer) return;
    cb_push(temp_buffer, temp);
}

float temp_history_get_average(void) {
    if (!temp_buffer || cb_is_empty(temp_buffer)) return 0.0f;
    
    float sum = 0;
    int count = cb_count(temp_buffer);
    float temp;
    
    // Temporary buffer for iteration
    CircularBuffer *temp = cb_create(count);
    if (!temp) return 0.0f;
    
    // Calculate sum while preserving buffer
    for (int i = 0; i < count; i++) {
        cb_pop(temp_buffer, &temp);
        sum += temp;
        cb_push(temp, temp);
    }
    
    // Restore original buffer
    CircularBuffer *swap = temp_buffer;
    temp_buffer = temp;
    temp = swap;
    
    cb_destroy(temp);
    return sum / count;
}

void temp_history_cleanup(void) {
    cb_destroy(temp_buffer);
    temp_buffer = NULL;
}

/*
 * Reusing the same circular buffer for a different purpose
 */

// event_logger.h
#ifndef EVENT_LOGGER_H
#define EVENT_LOGGER_H

typedef enum {
    EVENT_TEMP_HIGH,
    EVENT_TEMP_LOW,
    EVENT_SYSTEM_START,
    EVENT_SYSTEM_STOP
} EventType;

void event_logger_init(int capacity);
void event_logger_record(EventType event);
void event_logger_dump(void);
void event_logger_cleanup(void);

#endif

// event_logger.c
#include "event_logger.h"
#include "circular_buffer.h"
#include <stdio.h>

static CircularBuffer *event_buffer;

void event_logger_init(int capacity) {
    event_buffer = cb_create(capacity);
}

void event_logger_record(EventType event) {
    if (!event_buffer) return;
    cb_push(event_buffer, (float)event);
}

void event_logger_dump(void) {
    if (!event_buffer) return;
    
    int count = cb_count(event_buffer);
    float event;
    
    // Temporary buffer for iteration
    CircularBuffer *temp = cb_create(count);
    if (!temp) return;
    
    printf("Event Log:\n");
    for (int i = 0; i < count; i++) {
        cb_pop(event_buffer, &event);
        switch ((EventType)event) {
            case EVENT_TEMP_HIGH: printf("  High temperature alert\n"); break;
            case EVENT_TEMP_LOW: printf("  Low temperature alert\n"); break;
            case EVENT_SYSTEM_START: printf("  System started\n"); break;
            case EVENT_SYSTEM_STOP: printf("  System stopped\n"); break;
        }
        cb_push(temp, event);
    }
    
    // Restore original buffer
    CircularBuffer *swap = event_buffer;
    event_buffer = temp;
    temp = swap;
    
    cb_destroy(temp);
}

void event_logger_cleanup(void) {
    cb_destroy(event_buffer);
    event_buffer = NULL;
}
```

```c
/*
 * Bad example with duplicated buffer implementations
 */

// temperature_history.h
#ifndef TEMPERATURE_HISTORY_H
#define TEMPERATURE_HISTORY_H

void temp_history_init(int capacity);
void temp_history_record(float temp);
float temp_history_get_average(void);
void temp_history_cleanup(void);

#endif

// temperature_history.c
#include <stdlib.h>

typedef struct {
    float *buffer;
    int head;
    int tail;
    int size;
    int capacity;
} TempBuffer;

static TempBuffer *temp_buffer;

static bool is_empty(const TempBuffer *tb) {
    return tb->size == 0;
}

static bool is_full(const TempBuffer *tb) {
    return tb->size == tb->capacity;
}

void temp_history_init(int capacity) {
    temp_buffer = malloc(sizeof(TempBuffer));
    temp_buffer->buffer = malloc(capacity * sizeof(float));
    temp_buffer->head = 0;
    temp_buffer->tail = 0;
    temp_buffer->size = 0;
    temp_buffer->capacity = capacity;
}

void temp_history_record(float temp) {
    if (!temp_buffer || is_full(temp_buffer)) return;
    
    temp_buffer->buffer[temp_buffer->head] = temp;
    temp_buffer->head = (temp_buffer->head + 1) % temp_buffer->capacity;
    temp_buffer->size++;
}

float temp_history_get_average(void) {
    if (!temp_buffer || is_empty(temp_buffer)) return 0.0f;
    
    float sum = 0;
    for (int i = 0; i < temp_buffer->size; i++) {
        int idx = (temp_buffer->tail + i) % temp_buffer->capacity;
        sum += temp_buffer->buffer[idx];
    }
    return sum / temp_buffer->size;
}

void temp_history_cleanup(void) {
    if (temp_buffer) {
        free(temp_buffer->buffer);
        free(temp_buffer);
    }
}

/*
 * Duplicated buffer implementation for events
 */

// event_logger.h
#ifndef EVENT_LOGGER_H
#define EVENT_LOGGER_H

typedef enum {
    EVENT_TEMP_HIGH,
    EVENT_TEMP_LOW,
    EVENT_SYSTEM_START,
    EVENT_SYSTEM_STOP
} EventType;

void event_logger_init(int capacity);
void event_logger_record(EventType event);
void event_logger_dump(void);
void event_logger_cleanup(void);

#endif

// event_logger.c
#include <stdlib.h>
#include <stdio.h>

typedef struct {
    EventType *buffer;
    int head;
    int tail;
    int size;
    int capacity;
} EventBuffer;

static EventBuffer *event_buffer;

static bool event_is_empty(const EventBuffer *eb) {
    return eb->size == 0;
}

static bool event_is_full(const EventBuffer *eb) {
    return eb->size == eb->capacity;
}

void event_logger_init(int capacity) {
    event_buffer = malloc(sizeof(EventBuffer));
    event_buffer->buffer = malloc(capacity * sizeof(EventType));
    event_buffer->head = 0;
    event_buffer->tail = 0;
    event_buffer->size = 0;
    event_buffer->capacity = capacity;
}

void event_logger_record(EventType event) {
    if (!event_buffer || event_is_full(event_buffer)) return;
    
    event_buffer->buffer[event_buffer->head] = event;
    event_buffer->head = (event_buffer->head + 1) % event_buffer->capacity;
    event_buffer->size++;
}

void event_logger_dump(void) {
    if (!event_buffer || event_is_empty(event_buffer)) return;
    
    printf("Event Log:\n");
    for (int i = 0; i < event_buffer->size; i++) {
        int idx = (event_buffer->tail + i) % event_buffer->capacity;
        switch (event_buffer->buffer[idx]) {
            case EVENT_TEMP_HIGH: printf("  High temperature alert\n"); break;
            case EVENT_TEMP_LOW: printf("  Low temperature alert\n"); break;
            case EVENT_SYSTEM_START: printf("  System started\n"); break;
            case EVENT_SYSTEM_STOP: printf("  System stopped\n"); break;
        }
    }
}

void event_logger_cleanup(void) {
    if (event_buffer) {
        free(event_buffer->buffer);
        free(event_buffer);
    }
}
```

### Data Abstraction
Data abstraction involves designing components with a clean separation between their public interface and internal implementation. The public interface—typically defined in header files—includes type definitions, function declarations and constants that users need, while the implementation resides in source files or internal headers. 

This separation allows developers to modify or optimize the underlying implementation without affecting dependent code. For example, a `collection` abstraction might expose simple operations like add, remove and search, while internally it could be backed by an array, tree or graph—whatever best suits performance needs. Proper abstraction promotes loose coupling, better modularity and reduces compile-time dependecies. For clarity and ease of use, headers should be self-contained by including any dependencies they require. 

Lets take a look at a simple example to illustrate data abstraction using a `collection` module:

```c
// collection.h
#ifndef COLLECTION_H
#define COLLECTION_H

typedef struct Collection Collection;

Collection* collection_create();
void collection_add(Collection* c, int value);
int  collection_contains(Collection* c, int value);
void collection_destroy(Collection* c);

#endif // COLLECTION_H
```

```c
// collection.c
#include "collection.h"
#include <stdlib.h>

struct Collection {
    int* data;
    int size;
    int capacity;
};

Collection* collection_create() {
    Collection* c = malloc(sizeof(Collection));
    c->capacity = 10;
    c->size = 0;
    c->data = malloc(sizeof(int) * c->capacity);
    return c;
}

void collection_add(Collection* c, int value) {
    if (c->size >= c->capacity) {
        c->capacity *= 2;
        c->data = realloc(c->data, sizeof(int) * c->capacity);
    }
    c->data[c->size++] = value;
}

int collection_contains(Collection* c, int value) {
    for (int i = 0; i < c->size; i++) {
        if (c->data[i] == value) return 1;
    }
    return 0;
}

void collection_destroy(Collection* c) {
    free(c->data);
    free(c);
}
```

### Opaque Types
Opaque types are powerful tool in C for enforcing data abstraction by hiding the internal representation of a data structure from external users. This is commonly done using incomplete types, where a `struct` is forward-declared in a public header file without revealing its internal fields. This means the user of the type can only manipulate it through a pointer and cannot directly access or modify its internal. As a result, all interactions with the data must go through a well-defined public API, reducing the risk of misuse and accidental dependencies on internal details that may change in the future. 

This separation promotes **encapsulation**, helping to decouple the implementation from its interface. The actual structure definition is placed in a private header or source file used only by the implementation module. This design allows for flexibility in evolving the internal data layout or logic without breaking code that relies on the abstraction. For instance, a collection library might expose a `collection_type` in the API but keep its fields hidden internally. Users can still add or remove items via provided functions but remain unaware of whether the collection uses a linked list, array, or another structure. This results in more maintainable, modular, and robust code.

Here is an example how opaque type works:
- collection.h (public header which is visible to users):
```c
// Forward declaration (opaque type)
typedef struct collection_type collection_type;

// API function declarations
int create_collection(collection_type **col);
void destroy_collection(collection_type *col);
int add_to_collection(collection_type *col, const void *data, size_t size);
int remove_from_collection(collection_type *col, const void *data, size_t size);
```
- collection_internal.h (private header which is visible only to the implementation)
```c
struct node_type {
    void *data;
    size_t size;
    struct node_type *next;
};

struct collection_type {
    size_t num_elements;
    struct node_type *head;
};

```
- collection.c (source file which implements the logic)
```c
#include "collection.h"
#include "collection_internal.h"
#include <stdlib.h>
#include <string.h>

int create_collection(collection_type **col) {
    *col = malloc(sizeof(collection_type));
    if (!*col) return -1;
    (*col)->num_elements = 0;
    (*col)->head = NULL;
    return 0;
}

void destroy_collection(collection_type *col) {
    struct node_type *current = col->head;
    while (current) {
        struct node_type *next = current->next;
        free(current->data);
        free(current);
        current = next;
    }
    free(col);
}

```

With the level of setup and organizations, keeps `collection_type`'s structure hidden from any user that includes `collection.h`, while still allowing safe access to its functionality via a clean API. 
