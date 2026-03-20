# Course — Behavioral Spec

## Contracts

```
invariant: title is present
invariant: description is present (for published courses)
invariant: price_cents >= 0
invariant: currency is present
invariant: max_students > 0 (when present)
invariant: status in ["draft", "published", "archived"]

# PublishCourse
pre:  status == "draft"
pre:  description is not blank
pre:  has at least 1 module
pre:  each module has at least 1 lesson
pre:  organization has not exceeded max_courses limit
post: status == "published"
post: published_at is set to current time

# ArchiveCourse
pre:  status in ["draft", "published"]
post: status == "archived"

# UpdateCourse
pre:  status != "archived" (archived courses are immutable)

# DestroyCourse
pre:  no enrollments exist (any status)
```

## Authorization

| Action   | admin | instructor      | student   | guest     |
|----------|-------|-----------------|-----------|-----------|
| index    | ✓     | ✓               | published | published |
| show     | ✓     | own OR published| published | published |
| create   | ✓     | ✓               | ✗         | ✗         |
| update   | ✓     | own             | ✗         | ✗         |
| destroy  | ✓     | own AND draft   | ✗         | ✗         |
| publish  | ✓     | own             | ✗         | ✗         |
| archive  | ✓     | own             | ✗         | ✗         |

own for instructor: `course.instructor_id == current_user.id`

## Lifecycle

```
[draft] --publish--> [published] --archive--> [archived]
   \                                            /
    `---archive--> [archived]

publish: draft → published, guard: description + modules + lessons + org limit
archive: draft|published → archived, no guard

Terminal: archived (no outgoing transitions)
```

## Scenarios

```
Given course with 0 modules
When instructor calls POST /courses/:id/publish
Then 422, errors: ["At least one module required"]

Given course with 1 module that has 0 lessons
When instructor calls POST /courses/:id/publish
Then 422, errors: ["Each module must have at least one lesson"]

Given course with empty description
When instructor calls POST /courses/:id/publish
Then 422, errors: ["Description can't be blank"]

Given organization on free plan with 3 published courses
When instructor calls POST /courses/:id/publish (4th course)
Then 422, errors: ["Organization course limit reached"]

Given published course
When instructor calls DELETE /courses/:id
Then 403 (instructor can only delete own draft courses)

Given course with 2 enrollments
When admin calls DELETE /courses/:id
Then 422, errors: ["Cannot delete course with enrollments"]

Given archived course
When instructor calls PATCH /courses/:id with {title: "New Title"}
Then 422, errors: ["Archived courses are immutable"]

Given course owned by instructor_1
When instructor_2 calls PATCH /courses/:id
Then 403 (not own course)

Given draft course with description, 1 module, 1 lesson
When instructor calls POST /courses/:id/publish
Then 200, status == "published", published_at is set
And _links includes "archive" but not "publish"
```

## Side Effects

```
@on(publish):  set published_at = Time.current
@on(publish):  notify instructor "Course {title} published"
@on(create):   set instructor_id = current_user.id (if role is instructor)
@on(create):   set status = "draft", enrollments_count = 0
```
