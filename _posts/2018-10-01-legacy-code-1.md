---
layout: post
title:  "레거시 코드 활용 전략 - 서브클래스화와 메소드 재정의"
date:   2018-10-01 22:43:00 +0900
categories: learning-tech
tags: [TDD, Refactoring, 테스트 주도 개발, 리팩토링, legacy code]
---

`서브클래스화와 메소드 재정의` 기법은 객체 지향 프로그램에서 의존 관계를 제거하는 데 사용되는 주요 기법이다. 주된 아이디어는 관심 없는 동작을 무효화하거나 관심 있는 동작에 접근하기 위해 테스트 관점에서 상속을 이용하는 것이다.

```java
class MessageForwarder {
    private Message createForwardMessage(Session session, Message message) throws MessagingException, IOException {
        MimeMessage forward = new MimeMessage(session);
        forward.setFrom(getFromAddress(message));
        forward.setReplyTo(
            new Address[] {
                new InternetAddress(listAddress) 
            }
        );
        forward.addRecipients(Message.RecipientType.TO, listAddress);
        forward.addRecipients(Message.RecipientType.BCC, getMailListAddresses());
        forward.setSubject(transformedSubject(message.getSubject()));
        forward.setSentDate(message.getSentDate());
        forward.addHeader(LOOP_HEADER, listAddress);
        buildForwardContent(message, forward);
        return forward;
    }
    ...
}
```

`MessageForwarder` 클래스 안에 여러 메소드가 있는데, 그 중 public 메소드 하나가 위 `createForwardMessage` 메소드를 내부에서 호출한다. 그 public 메소드는 `createForwardMessage` 를 호출하고 `MimeMessage` 객체를 반환 받는다. 이를 내가 정의한 `FakeMessage` 로 반환하도록하여 테스트를 하려면 어떻게 해야할까?

`createForwardMessage` 메소드를 `protected` 로 만들고 테스트 코드에서 메소드를 재정의한다. 그리고 테스트한다.
```java
class TestingMessageForwarder extends MessageForwarder {
    protected Message createForwardMessage(Session session, Message message) {
        Message forward = new FakeMessage(message);
        return forward;
    }
}
```

이처럼 테스트 관점에서 상속(메소드 재정의)를 이용하여 의존 관계를 끊어낼 수 있다. 테스트 코드에서 의도적으로 원하는 상황을 만들어줄 수 있다.

#### 참고
- https://book.naver.com/bookdb/book_detail.nhn?bid=14032002
- https://www.youtube.com/watch?v=_NnElPO5BU0