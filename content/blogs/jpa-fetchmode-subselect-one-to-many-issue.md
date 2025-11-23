---
title: "The Issue with FetchMode.SUBSELECT and @OneToMany Mappings in Hibernate and JPA"
date: 2020-11-22T00:00:00+05:30
draft: false
tags: ["hibernate", "jpa", "java", "subselect", "one to many"]
image: /images/blogs/banner-jpa-fetchmode-subselect-one-to-many-issue.jpg
---

This article is about a performance issue of `@OneToMany` mappings when used in conjunction with `FetchMode.SUBSELECT`.


**If not used properly, it can even load an entire table in memory**.


Lets consider the following entity relationship diagram

![Entity Relationship Diagram](/images/blogs/jpa-fetchmode-subselect-one-to-many-issue-entity-rel.webp)

A department will have multiple employees. So we can have a one-to-many mapping. We have `dept_id` column in `employee` table, which is a foreign key referring to id column in `dept` table.


Here's the SQL for creating the tables

```sql
CREATE TABLE IF NOT EXISTS `dept` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  PRIMARY KEY (`id`))
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8mb4;

CREATE TABLE IF NOT EXISTS `employee` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  `dept_id` INT NOT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_employee_dept_id_idx` (`dept_id` ASC),
  CONSTRAINT `fk_employee_dept_id`
    FOREIGN KEY (`dept_id`)
    REFERENCES `dept` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8mb4;

INSERT INTO `dept` (`name`) VALUES ('Engineering');
-- id = 1
INSERT INTO `dept` (`name`) VALUES ('HR');
-- id = 2
INSERT INTO `dept` (`name`) VALUES ('Finance');
-- id = 3

INSERT INTO `employee` (`name`, `dept_id`) VALUES ('John Smith', 1);
INSERT INTO `employee` (`name`, `dept_id`) VALUES ('Reily Clark', 1);
INSERT INTO `employee` (`name`, `dept_id`) VALUES ('Jane Smith', 2);
INSERT INTO `employee` (`name`, `dept_id`) VALUES ('Frank Evans', 2);
INSERT INTO `employee` (`name`, `dept_id`) VALUES ('Adam Baker', 3);
INSERT INTO `employee` (`name`, `dept_id`) VALUES ('Irvin Jones', 3);
```

The entity classes are given below

```java
@Entity
@Table(name = "dept")
@Getter
@Setter
public class Dept {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;
    @OneToMany(
        mappedBy = "dept",
        cascade = CascadeType.ALL,
        orphanRemoval = true
    )
    @Fetch(FetchMode.SUBSELECT)
    private List<Employee> employees;
}
@Entity
@Table(name = "employee")
@Getter
@Setter
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;
    @ManyToOne(fetch = FetchType.LAZY)
    private Dept dept;
    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof Employee)) {
            return false;
        }
        return id != null && id.equals(((Employee) o).getId());
    }
    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

### The Performance Hit

Lets say we are fetching a paginated list of departments. We may use the following `PagingAndSortingRepository` method for getting the paginated list.

```java
final Page<Dept> depts = deptRepository.findAll(pageable);

for (Dept dept : depts) {
    final List<Employee> employees = dept.getEmployees();
    System.out.println(employees);
}
```

For the above code snippet Hibernate will issue the following SQLs

```text
Hibernate: select ... from dept dept0_ limit ?
Hibernate: select ... from employee employees0_ where employees0_.dept_id in (select dept0_.id from dept dept0_)
```

The first query loads the first 10 departments. But the second query has a serious problem… **The limit clause is missing in the subquery!! This will load the entire employee table in memory just for getting a few records**.

### Reason

When Hibernate loads the parent entities, it remembers the query used to load the same. The parent-query is then used as a subquery when Hibernate tries to load the child entities. While Hibernate considers the `where` conditions, it does not remember the limit and offset clauses in the parent-query. If the child table has millions of records, the performance hit will be severe.

### Conclusion

`FetchMode.SUBSELECT` is useful as it solves the `N+1` query[¹](https://vladmihalcea.com/n-plus-1-query-problem/) issue of `FetchMode.SELECT`. But `FetchMode.SUBSELECT` leads to increased memory usage as the child table size increases. In such cases `FetchMode.SELECT` with `@BatchSize` performs better. But in reality, if the child table is large, its better to avoid `@OneToMany` mapping, since `@ManyToOne` might be just enough[²](https://vladmihalcea.com/the-best-way-to-map-a-onetomany-association-with-jpa-and-hibernate/).

---

### References:

[1] https://vladmihalcea.com/n-plus-1-query-problem/

[2] https://vladmihalcea.com/the-best-way-to-map-a-onetomany-association-with-jpa-and-hibernate/