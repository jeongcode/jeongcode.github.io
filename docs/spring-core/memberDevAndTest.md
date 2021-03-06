---
layout: default
title: 회원 개발 및 테스트
grand_parent: 스프링 핵심
parent: 스프링 핵심 원리 이해
nav_order: 1
---
## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
### **Grade**
```java
package hello.core.member;

public enum Grade {
    BASIC,
    VIP
}
```
### **Member**
```java
public class Member {
    private Long id;
    private String name;
    private Grade grade;

    public Member(Long id, String name, Grade grade) {
        this.id = id;
        this.name = name;
        this.grade = grade;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Grade getGrade() {
        return grade;
    }

    public void setGrade(Grade grade) {
        this.grade = grade;
    }
}
```

### **MemberRepository , Service**
```java
public interface MemberRepository {
    void save(Member member);
    Member findById(Long memberId);
}

public interface MemberService {
    void join(Member member);
    Member findMember(Long memberId);
}
```

### **MemoryMemberRepository**
```java
public class MemoryMemberRepository implements MemberRepository{

    private static Map<Long , Member> store = new HashMap<>();

    @Override
    public void save(Member member) {
        store.put(member.getId() , member);
    }

    @Override
    public Member findById(Long memberId) {
        return store.get(memberId);
    }
}
```

### **MemberServiceImpl**
```java
public class MemberServiceImpl implements MemberService{

    private final MemberRepository memberRepository = new MemoryMemberRepository();

    @Override
    public void join(Member member) {
        memberRepository.save(member);
    }

    @Override
    public Member findMember(Long memberId) {
        return memberRepository.findById(memberId);
    }
}
```

### **MemberApp(순수 자바 코드로 테스트)**
```java
// 순수 자바 코드로 테스트
public class MemberApp {
    public static void main(String[] args) {
        MemberService memberService = new MemberServiceImpl();
        // Ctrl + Alt + V - 반환 되는 값에 맞는 인스턴스를 자동 기입
        Member member = new Member(1L, "memberA", Grade.VIP);
        memberService.join(member);

        Member findMember = memberService.findMember(1L);
        System.out.println("new Member =" + member.getName());
        System.out.println("find Member =" + findMember.getName());
    }
}
```
### **MemberServiceTest**
```java
package hello.core.member;

import org.assertj.core.api.Assertions;
import org.junit.jupiter.api.Test;

public class MemberServiceTest {

    MemberService memberService = new MemberServiceImpl();

    @Test
    void join(){
        //given
        Member member = new Member(1L , "memberA" , Grade.VIP);

        //when
        memberService.join(member);
        Member findMember = memberService.findMember(1L);

        //then
        Assertions.assertThat(member).isEqualTo(findMember);
    }
}
```

### 🚨 **회원 도메인 설계의 문제점**
-   다른 저장소로 변경할 때 OCP 원칙을 잘 준수할까요?
-   DIP를 잘 지키고 있을까요?
-   **의존관계가 인터페이스 뿐만 아니라 구현까지 모두 의존하는 문제점이 있음**
