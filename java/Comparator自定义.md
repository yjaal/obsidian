
```java
  
public class Student{

	private int id;
	private String name;
}

// 基本实现

Comparator<Student> comp = new Comparator<Student>(){

	@Override
	public int compare(Student s1, Student s2){
	return s1.getId() - s2.getId();
	}
}

// 流式实现
Comparator.comparing(s -> (s1.getId - s2.getId) + (s1.getName() == s2.getName ? 0 : 1));
```