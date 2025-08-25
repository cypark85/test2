# ai_test_agent Database Schema (PostgreSQL)

- Database: `ai_test_agent`
- Schema: `public`
- Owner: `prod_user`
- Generated: current snapshot from running DB

## Table: exam_definitions

| Column                 | Type                         | Nullable | Default                                   | Description |
|------------------------|------------------------------|----------|-------------------------------------------|-------------|
| id                     | integer                      | no       | nextval('exam_definitions_id_seq')        | Primary key |
| exam_name              | varchar(255)                 | no       |                                           | 시험 이름 |
| problem_bank_set_id    | integer                      | no       |                                           | FK to `problem_bank_sets.id` |
| total_questions_count  | integer                      | no       |                                           | 총 문항 수 |
| total_steps_count      | integer                      | no       |                                           | 총 스텝(라운드) 수 |
| questions_per_step     | integer                      | no       |                                           | 스텝당 문항 수 |
| exam_description       | text                         | yes      |                                           | 시험 설명 |
| is_active              | boolean                      | yes      | true                                      | 활성화 여부 |
| created_timestamp      | timestamp without time zone  | yes      | CURRENT_TIMESTAMP                          | 생성 시각 |
| last_updated_timestamp | timestamp without time zone  | yes      | CURRENT_TIMESTAMP                          | 업데이트 시각 |

- Indexes: `exam_definitions_pkey(id)`, `idx_exam_definitions_active(is_active)`, `idx_exam_definitions_bank_set(problem_bank_set_id)`
- FKs: `problem_bank_set_id → problem_bank_sets(id) ON DELETE CASCADE`
- Referenced by: `exam_sessions.exam_definition_id`

---

## Table: exam_sessions

| Column                      | Type                         | Nullable | Default                                   | Description |
|----------------------------|------------------------------|----------|-------------------------------------------|-------------|
| id                          | integer                      | no       | nextval('exam_sessions_id_seq')           | Primary key |
| student_id                  | integer                      | no       |                                           | FK to `students.id` |
| current_step_number         | integer                      | yes      | 1                                         | 현재 스텝 번호 |
| session_status              | session_status_enum          | yes      | 'in_progress'                             | 세션 상태 |
| session_started_timestamp   | timestamp without time zone  | yes      |                                           | 시작 시각 |
| session_completed_timestamp | timestamp without time zone  | yes      |                                           | 완료 시각 |
| created_timestamp           | timestamp without time zone  | yes      | CURRENT_TIMESTAMP                          | 생성 시각 |
| last_updated_timestamp      | timestamp without time zone  | yes      | CURRENT_TIMESTAMP                          | 업데이트 시각 |
| exam_definition_id          | integer                      | yes      |                                           | FK to `exam_definitions.id` |
| report                      | jsonb                        | yes      |                                           | 세션 리포트(서버 생성 데이터) |

- Indexes: `exam_sessions_pkey(id)`, `idx_exam_sessions_definition(exam_definition_id)`, `idx_exam_sessions_status(session_status)`, `idx_exam_sessions_student(student_id)`
- FKs: `exam_definition_id → exam_definitions(id) ON DELETE CASCADE`, `student_id → students(id) ON DELETE CASCADE`
- Referenced by: `student_problem_predictions.exam_session_id`

---

## Table: problem_bank_sets

| Column                 | Type                         | Nullable | Default                                   | Description |
|------------------------|------------------------------|----------|-------------------------------------------|-------------|
| id                     | integer                      | no       | nextval('problem_bank_sets_id_seq')       | Primary key |
| set_name               | varchar(255)                 | no       |                                           | 세트 이름 |
| set_description        | text                         | yes      |                                           | 설명 |
| is_currently_active    | boolean                      | yes      | true                                      | 활성화 여부 |
| created_timestamp      | timestamp without time zone  | yes      | CURRENT_TIMESTAMP                          | 생성 시각 |
| last_updated_timestamp | timestamp without time zone  | yes      | CURRENT_TIMESTAMP                          | 업데이트 시각 |

- Indexes: `problem_bank_sets_pkey(id)`, `idx_problem_bank_sets_active(is_currently_active)`
- Referenced by: `exam_definitions.problem_bank_set_id`, `problems.problem_bank_set_id`

---

## Table: problems

| Column                   | Type                        | Nullable | Default                          | Description |
|--------------------------|-----------------------------|----------|----------------------------------|-------------|
| id                       | integer                     | no       | nextval('problems_id_seq')       | Primary key |
| problem_bank_set_id      | integer                     | no       |                                  | FK to `problem_bank_sets.id` |
| problem_type             | problem_type_enum           | no       |                                  | 문제 유형(단어/문장/문법) |
| question_content         | text                        | no       |                                  | 문제 본문 |
| correct_answer_content   | text                        | no       |                                  | 정답 텍스트 |
| type_specific_metadata   | jsonb                       | yes      |                                  | 유형별 메타정보(JSON) — 예: 문장/단어 `level(1-10)`, 문법 `major_category`, `subcategory`, `level(상/중/하)`, `meta_information`, `related_diagnosis` |
| created_timestamp        | timestamp without time zone | yes      | CURRENT_TIMESTAMP                 | 생성 시각 |
| last_updated_timestamp   | timestamp without time zone | yes      | CURRENT_TIMESTAMP                 | 업데이트 시각 |
| problem_sequence_in_set  | integer                     | yes      |                                  | 세트 내 순번(1부터) |

- Indexes: `problems_pkey(id)`, `idx_problems_bank_set(problem_bank_set_id)`, `idx_problems_bank_set_sequence(problem_bank_set_id, problem_sequence_in_set)`, `idx_problems_metadata USING GIN(type_specific_metadata)`, `idx_problems_type(problem_type)`, `uk_problem_bank_set_sequence UNIQUE(problem_bank_set_id, problem_sequence_in_set)`
- FKs: `problem_bank_set_id → problem_bank_sets(id) ON DELETE CASCADE`
- Referenced by: `student_problem_predictions.problem_id`

---

## Table: student_problem_predictions

| Column                        | Type                        | Nullable | Default                                        | Description |
|-------------------------------|-----------------------------|----------|------------------------------------------------|-------------|
| id                            | integer                     | no       | nextval('student_problem_predictions_id_seq')  | Primary key |
| exam_session_id               | integer                     | no       |                                                | FK to `exam_sessions.id` |
| problem_id                    | integer                     | no       |                                                | FK to `problems.id` |
| ai_predicted_correctness      | boolean                     | yes      |                                                | AI 사전 예측 정오 |
| ai_prediction_confidence_score| numeric(3,2)                | yes      |                                                | 예측 신뢰도 |
| ai_prediction_reasoning_text  | text                        | yes      |                                                | 예측 근거 텍스트 |
| assigned_step_number          | integer                     | yes      |                                                | 배정된 스텝 번호 |
| position_in_step              | integer                     | yes      |                                                | 스텝 내 위치 |
| problem_presented_timestamp   | timestamp without time zone | yes      |                                                | 문제 제시 시간 |
| student_has_answered          | boolean                     | yes      | false                                          | 학생 답변 여부 |
| student_actual_answer         | text                        | yes      |                                                | 학생 답변 |
| answer_submitted_timestamp    | timestamp without time zone | yes      |                                                | 제출 시각 |
| response_time_in_seconds      | integer                     | yes      |                                                | 응답 시간(초) |
| ai_graded_correctness         | boolean                     | yes      |                                                | AI 채점 정오 |
| ai_grading_reasoning_text     | text                        | yes      |                                                | 채점 근거 텍스트 |
| ai_grading_confidence_score   | numeric(3,2)                | yes      |                                                | 채점 신뢰도 |
| ai_grading_timestamp          | timestamp without time zone | yes      |                                                | 채점 시각 |
| student_confidence_level      | smallint                    | yes      |                                                | 자신감(1~5) |
| student_confidence_reasoning  | text                        | yes      |                                                | 자신감 근거 |

- Constraints: `CHECK student_confidence_level BETWEEN 1 AND 5`
- Indexes: `student_problem_predictions_pkey(id)`, `idx_predictions_exam_session(exam_session_id)`, `idx_predictions_problem(problem_id)`, `idx_predictions_step(assigned_step_number)`, `idx_predictions_session_step(exam_session_id, assigned_step_number)`, `idx_predictions_answered(student_has_answered)`, `idx_predictions_ai_predicted(ai_predicted_correctness)`, `idx_predictions_ai_graded(ai_graded_correctness)`, `idx_predictions_ai_comparison(ai_predicted_correctness, ai_graded_correctness)`, `unique_session_problem UNIQUE(exam_session_id, problem_id)`
- FKs: `exam_session_id → exam_sessions(id) ON DELETE CASCADE`, `problem_id → problems(id) ON DELETE CASCADE`

---

## Table: students

| Column                 | Type                         | Nullable | Default                          | Description |
|------------------------|------------------------------|----------|----------------------------------|-------------|
| id                     | integer                      | no       | nextval('students_id_seq')       | Primary key |
| student_name           | varchar(100)                 | no       |                                  | 학생명 |
| unique_student_id      | uuid                         | no       |                                  | 외부 식별용 UUID (UNIQUE) |
| current_grade_level    | smallint                     | yes      |                                  | 학년/레벨 |
| created_timestamp      | timestamp without time zone  | yes      | CURRENT_TIMESTAMP                 | 생성 시각 |
| last_updated_timestamp | timestamp without time zone  | yes      | CURRENT_TIMESTAMP                 | 업데이트 시각 |

- Indexes/Constraints: `students_pkey(id)`, `idx_students_grade(current_grade_level)`, `idx_students_unique_id UNIQUE(unique_student_id)`, `UNIQUE(unique_student_id)`
- Triggers: `trigger_generate_student_uuid BEFORE INSERT → generate_student_uuid()`
- Referenced by: `exam_sessions.student_id`

---

## Table: test_groups (custom)

| Column                    | Type                          | Nullable | Default            | Description |
|---------------------------|-------------------------------|----------|--------------------|-------------|
| id                        | uuid                          | no       | gen_random_uuid()  | Primary key |
| user_id                   | uuid                          | no       |                    | 앱 유저 UUID |
| word_test_session_id      | integer                       | yes      |                    | 단어 시험 `exam_sessions.id` |
| sentence_test_session_id  | integer                       | yes      |                    | 문장 시험 `exam_sessions.id` |
| grammar_test_session_id   | integer                       | yes      |                    | 문법 시험 `exam_sessions.id` |
| created_at                | timestamp with time zone      | no       | now()              | 생성 시각 |

- Indexes: `test_groups_pkey(id)`, `idx_test_groups_user_id(user_id)`, `idx_test_groups_word_session(word_test_session_id)`, `idx_test_groups_sentence_session(sentence_test_session_id)`, `idx_test_groups_grammar_session(grammar_test_session_id)`, `idx_test_groups_created_at(created_at)`

---

## Notes
- Enums
  - `session_status_enum`: includes at least `'in_progress'`; additional values may exist in DB.
  - `problem_type_enum`: problem types used by workflows (e.g., word, sentence, grammar).
- Timestamps use server time; consider timezone handling on client.
- JSONB fields are indexed where necessary for query performance. 