
**INTRO:** <br>
A java program that can search through and perform queries on a school database using JDBC and MYSQL.

![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/51fb1447-cf17-446c-a2fa-21dbaadc0c1d)


**HOW IT WORKS AND IMPLEMENTATION:**

When running this program a menu will be prompted showing the users a list of options to pick from. These options could do specific queries like searching the database to find a specific student, get department/class statistics, or perform queries that the user input. 

This program is very simple, the main principle to know is that you execute queries by passing it to the resultset which is a built-in function to traverse through the database.
 
(User inputs 1-4)
The searchStudents() function performs user inputs 1-4, it has if statements that changes the query depending on what option is inputted. Here is a simple explanation of how my code searches for students. I first use a broad query to get the name of the student and their id. While in the loop, I get the id of the current student and execute another query within that loop that is more specific where the id is equal to the current student. This is used to get the gpa, major, and minor etc. Think of it like a 2d array/nested loop. 

(User input 5)
The getDepartmentStats() function is used to count the number of students in the department and their average gpa. Similarly to how we get the gpa above, instead of a nested loop, we only execute a single query that joins the major and minors table together. The reason it's not a nested loop this time is because we are treating it as one big student. A nested for loop is to get the gpa of the current student, whereas for the departments that is not necessary we just need to get whoevers grades and credits is in the specified department. Some things to note is that we used a left join on the majors and minors table because the student grade will not be calculated if they don’t have a minor or major, and a left join will fix that (tested), and also the student cannot major and minor in the department, we created a trigger to prevent that as mentioned in the design phase.

(User input 6)
The getClassStats() function counts the number of students that are currently taking a class and lists the grades of previous enrollees. This is very simple to do with a query that involves grouping the data and using a count aggregate function. 

(User input 7)
The getQuery() function execute a query that the user inputs, you can also input statements like inserts and it’ll work. It first checks whether the query starts with “select” (upper or lowercase doesn’t matter), else it will simply do an executeUpdate for the given statement. If it starts with “select”  I use resultsetmetadata to get the formatting of the table. ResultSetMetaData has built-in methods like .getColumnCount() which is pretty useful to use to print out the column names, and get the input.

![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/fcd7922d-64c2-4863-aa3e-6b5715c9c792)
![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/c9c21d53-96b8-4be0-a275-aa3e8526f7c2)
![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/d8704468-0ef6-49c9-8337-0553761902bd)
![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/94013280-92d2-434c-b7a1-6ddcc269ceae)
![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/4c138abd-1717-4c60-aa62-5241bbdb45d3)
![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/9d671269-b065-498c-844b-95ec685ad09c)

<hr>

**CREATING THE DATABASE**

**TABLE DESIGN:**

The following statements show the creation of the layout and structures of the database tables. As you can see it is very basic and includes some constraints to meet the requirements of the instructions. Read my comments below to see my thought process and assumptions behind that design.

CREATE TABLE departments ( <br>
	Name varchar(10), <br>
	campus varchar(10), <br>
	PRIMARY KEY (name) <br>
);
	
//Constraint to ensure id is 9 digits <br>
CREATE TABLE students ( <br>
	first_name varchar(30), <br>
	last_name varchar(30), <br>
	ID int PRIMARY KEY, <br>
CONSTRAINT id_length_check CHECK (LENGTH(id) = 9) <br>
);

CREATE TABLE classes ( <br>
	Name varchar(75), <br>
	Credits int, <br>
	Primary key (name) <br>
);


//constraint to ensure that there are no duplicate entries in the majors table <br>
//this design also assumes that students can have 0 majors, I did not add a not null constraint<br>

CREATE TABLE majors ( <br>
	Sid int, <br>
	Dname varchar(10), <br><br>
	Foreign key (sid) REFERENCES students (id),<br>
	Foreign key (dname) REFERENCES departments(name),<br>
	CONSTRAINT no_duplicates UNIQUE(sid,dname)	<br>
);

CREATE TABLE minors (<br>
	Sid int, <br>
	Dname varchar(10), <br>
	Foreign key (sid) REFERENCES students (id),<br>
	Foreign key (dname) REFERENCES departments(name),<br>
CONSTRAINT no_duplicates UNIQUE(sid,dname)	<br>
);

CREATE TABLE istaking ( <br>
	Sid int,<br>
	Name varchar(75),<br>
	Foreign key (sid) REFERENCES students (id),<br>
	Foreign key (name) REFERENCES classes (name),<br>
	CONSTRAINT no_duplicates UNIQUE (sid,name)<br>
);
		
//constraint make sure that theres no duplicate entries <br>
//this also stores records of student taking the same course (if they fail and have to re-take) <br>

CREATE TABLE hastaken ( <br>
	Sid int,<br>
	Name varchar(75),<br>
	Grade char(1) ,<br>
	Foreign key (sid) REFERENCES students (id),<br>
	Foreign key (name) REFERENCES classes (name),<br>
	CONSTRAINT no_duplicates UNIQUE (sid,name,grade)<br>
);

//create trigger to make sure students can’t major and minor in same departments <br>
//I tried to create assertion for this but sql doesn’t allow ( could be the workbench I’m using) <br>

DELIMITER // <br>
CREATE TRIGGER no_duplicate_major_trigger BEFORE INSERT ON minors<br>
FOR EACH ROW<br>
BEGIN<br>
    IF NEW.dname IN (<br>
   	 SELECT m.dname<br>
   	 FROM majors m<br>
   	 WHERE m.sid = NEW.sid)<br>
    THEN<br>
   	 SIGNAL SQLSTATE '45000'<br>
   		 SET MESSAGE_TEXT = 'ERROR WITH INSERTION: student already major in the same department';<br>
    END IF;<br>
END;//<br>
DELIMITER ;<br>

DELIMITER //<br>
CREATE TRIGGER no_duplicate_minor_trigger BEFORE INSERT ON majors<br>
FOR EACH ROW<br>
BEGIN<br>
    IF NEW.dname IN (<br>
   	 SELECT m.dname<br>
   	 FROM minors m<br>
   	 WHERE m.sid = NEW.sid)<br>
    THEN<br>
   	 SIGNAL SQLSTATE '45000'<br>
   		 SET MESSAGE_TEXT = 'ERROR WITH INSERTION: student already minor in the same department';<br>
    END IF;<br>
END;//<br>
DELIMITER ;<br>


**POPULATING TABLES:** 

I manually insert data directly into the database for smaller tables like classes and departments. However, for larger tables like the students table, I used tools like Mockaroo to generate random data in SQL format. To populate the 'hastaken' table with data, I used a random number generator in Java that was connected with JDBC. Each number generated was assigned a predefined class, grade, and other related information. I would loop this like 25 times for each students to ensure that I get a specific number of credits to make the student a senior, and then lower it to 20 times for juniors etc.

![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/f414c947-a8ab-4d0d-8c80-f8fe0468c7a7)
![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/5b146cb0-2a38-48a2-ba96-ea086cbc76f1)
![image](https://github.com/jnguyen02/SchoolJDBC/assets/111792553/86b3c4e9-3879-457e-a364-bee1c047c6ae)

