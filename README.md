# Architecture Proposal for Questionnaire Application

#### 1. **Overview**
The proposed architecture consists of a frontend for user interaction and a backend for handling business logic and data storage. The frontend and backend communicate via RESTful APIs.

#### 2. **Frontend**
- **Type**: Single Page Application (SPA)
- **Framework**: React, Angular, Vue
- **Deployment**: Cloud service like AWS Amplify, Vercel, or Netlify for easy scaling and global reach.

#### 3. **Backend**
- **Type**: RESTful API
- **Framework**: Node.js with Express.js or Python with FastAPI
- **Deployment**: AWS Lambda (Serverless) or Dockerized application on AWS ECS

#### 4. **Database**
- **Type**: Relational Database
- **Proposed Database**: PostgreSQL
- **Deployment**: Managed service like AWS RDS

### Database Schema

#### Tables

1. **Users**
   - `id`: UUID (Primary Key)
   - `username`: VARCHAR
   - `email`: VARCHAR
   - `created_at`: TIMESTAMP

2. **Products**
   - `id`: UUID (Primary Key)
   - `name`: VARCHAR
   - `description`: TEXT
   - `created_at`: TIMESTAMP

3. **Questionnaires**
   - `id`: UUID (Primary Key)
   - `title`: VARCHAR
   - `created_at`: TIMESTAMP
   - `max_products`: INT

4. **Questions**
   - `id`: UUID (Primary Key)
   - `questionnaire_id`: UUID (Foreign Key)
   - `text`: TEXT
   - `created_at`: TIMESTAMP

5. **Responses**
   - `id`: UUID (Primary Key)
   - `user_id`: UUID (Foreign Key)
   - `question_id`: UUID (Foreign Key)
   - `response_text`: TEXT
   - `response_time`: INTERVAL
   - `created_at`: TIMESTAMP

6. **SurveySessions**
   - `id`: UUID (Primary Key)
   - `user_id`: UUID (Foreign Key)
   - `questionnaire_id`: UUID (Foreign Key)
   - `start_time`: TIMESTAMP
   - `end_time`: TIMESTAMP (nullable)

7. **Metrics**
   - `questionnaire_id`: UUID (Primary Key)
   - `total_responses`: INT
   - `average_response_time`: INTERVAL
   - `average_completion_time`: INTERVAL
   - `products_issued`: INT

#### Indexes

- **Users**: Index on `email` for fast lookup.
- **Responses**: Composite index on `question_id` and `user_id` for efficient querying.
- **SurveySessions**: Index on `user_id` for session tracking.

### Transaction Atomicity

- **Where Necessary**:
  - When recording responses to ensure all parts of a response are recorded together.
  - When updating metrics to ensure consistency.

- **How to Achieve**:
  - Using database transactions in PostgreSQL. Example in Node.js using Sequelize:
    ```javascript
    const transaction = await sequelize.transaction();
    try {
      await Response.create(responseData, { transaction });
      await Metrics.update(metricsData, { transaction });
      await transaction.commit();
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
    ```

### Determining Questionnaire Completion Time

- **Method**:
  - Record the start time when the user begins the questionnaire.
  - Record the end time upon submission of the last question.
  - Calculate the difference between the end time and start time.
  - Example in Node.js:
    ```javascript
    const completionTime = endTime - startTime;
    ```

### Handling Unfinished Sessions

- **Method**:
  - Implement a session timeout mechanism. If a session exceeds a certain duration without completion, mark it as incomplete.
  - Periodically check for incomplete sessions and update their status.
  - Notify the user if they can resume the session or need to restart.

- **Implementation**:
  ```javascript
  const INCOMPLETE_THRESHOLD = 30 * 60 * 1000; // 30 minutes

  const checkIncompleteSessions = async () => {
    const incompleteSessions = await SurveySessions.findAll({
      where: {
        end_time: null,
        start_time: {
          [Op.lt]: new Date(Date.now() - INCOMPLETE_THRESHOLD)
        }
      }
    });

    for (const session of incompleteSessions) {
      // Mark session as incomplete
      session.end_time = new Date();
      await session.save();
    }
  };

  setInterval(checkIncompleteSessions, 10 * 60 * 1000); // Run every 10 minutes
  ```

### Dashboard Metrics

- **Data Collection**:
  - Count the number of users who answered each question using the Responses table.
  - Calculate the average response time and completion time from the Responses and SurveySessions tables.
  - Track the number of products issued via the Products table.

- **Presentation**:
  - Use a dashboard framework like Grafana or a custom-built dashboard with React and Chart.js.
  - Display metrics in real-time, updated periodically from the backend.