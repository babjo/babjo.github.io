---
layout: post
title:  "리팩토링 전략 - 적절하게 책임을 분배하자"
date:   2018-10-07 11:39:00 +0900
categories: learning-tech
tags: [refactoring, 리팩토링, TDD]
---

어느 회사든 들어가면, 처음부터 자신이 작성하는 코드는 드물어요. 이미 존재했던 프로젝트가 있으면 누군가가 작성해놓은 코드가 있는거죠. 업무를 받으면 그 코드부터 분석하고 변경점을 찾아서 수정하는 일이 많아요. 그런데, 그 코드가 읽기 어려운 경우가 종종 있는데, 제 경우는 코드를 먼저 깔끔하게 리팩토링을 먼저 진행하고 그 다음 업무를 진행해요. 언젠가는 갚아야하는 비용이니 미리 개선을 하는거죠. (물론 개선하는걸 좋아하긴해요.)

제가 생각하는 깔끔하는 코드는 **읽기 쉬운 코드**라 생각해요. 마치 영어 문장을 읽듯이 코드가 술술 읽히는거죠. 그렇게 코드를 리팩토링하려면, 먼저 먼저 미리 존재하는 코드(**레거시 코드** 라 부를게요)를 정확히 파악하는게 필요해요. 그 다음은 그 파악한 코드를 이 레거시 코드를 모르는 사람에게 설명한다고 했을 때, 간략히 한번 적어보고요. (이렇게 하는게 역할을 단순화해서 파악하는데 도움이 되요.) 그 다음에 이 레거시 코드가 하는 일(책임)들을 쭉 나열해는거에요. 그 뒤는 이를 다시 적절한 클래스에 재분배하는거죠. 요약하자면, "레거시 코드 분석하고 쪼개고 조합하고" 예요.

예를 들어볼게요. ('레거시 활용 전략' 이라는 책에서 나온 예에요. 조금 각색해서 사용했어요.) 예제지만, 코드가 숨숨 턱턱 막히는군요. 쭉 읽어보아요. (저는 미리 파악하면서 약간의 코멘트도 남겨놨어요.)

```java
import java.io.IOException;
import java.util.Properties;

import javax.mail.*;
import javax.mail.internet.*;

public class MailingListServer {
    public static final String SUBJECT_MARKER = "[list]";
    public static final String LOOP_HEADER = "X-Loop";

    public static void main(String[] args) {
        if (args.length != 8) {
            System.err.println("Usage: java MailingList <popHost> " +
                "<smtpHost> <pop3user> <pop3password> " +
                "<smtpuser> <smtppassword> <listAddress> " +
                "<replayinterval>");
            return;
        }
        HostInformation host = new HostInformation(
            args[0], args[1], args[2], args[3],
            args[4], args[5]);
        String listAddress = args[6];
        int interval = new Integer(args[7]).intValue();
        Roster roster = null;

        // roster.txt (설정 파일) 을 읽어옴
        try {
            roster = new FileRoster("roster.txt");
        } catch (Excetion e) {
            System.err.println("unable to open roster.txt");
            return;
        }
        try {
            do {
                try {
                    Properties properties = System.getProperties();
                    Session session = Session.getDefaultInstance(
                        properties, null);
                    Store store = session.getStore("pop3");
                    store.connect(host.pop3Host, -1, 
                        host.pop3User, host.pop3Password);
                    Folder defaultFolder = store.getDefaultFolder();
                    if (defaultFolder == null) {
                        System.err.prinln("Unable to open default folder");
                        return;
                    }
                    // pop3 host 에 연결해서 INBOX 폴더를 가져옴
                    Folder folder = defaultFolder.getFolder("INBOX");
                    if (folder == null) {
                        System.err.println("Unable to get: " + defaultFolder);
                        return;
                    }
                    folder.open(Folder.READ_WRITE);

                    // 처리?
                    process(host, listAddress, roster, session, store, folder);
                } catch (Exception e) {
                    System.err.println(e);
                    System.err.println("(retrying mail check)");
                }
                System.err.print(".");

                // 일정시간동안 쉬고 다시 시도함
                try { Thread.sleep(interval * 1000); }
                catch (InterruptedException e) {}
            } while(true);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void process(
        HostInformation host, String listAddress, Roster roster,
        Session session, Store store, Folder folder)
            throws MessagingException {
        try {
            if (folder.getMessageCount() != 0) {

                // 폴더에서 메세제들을 가져와 전송
                Message[] messages = folder.getMessages();
                doMessage(host, listAddress, roster, session,
                    folder, messages);
            }
        } catch (Exception e) {
            System.err.println("message handling error");
            e.printStackTrace(System.err);
        } finally {
            folder.close(true);
            store.close();
        }
    }

    private static void doMessage(
        HostInformation host,
        String listAddress,
        Roster roster,
        Session session,
        Folder folder,
        Message[] messages) throws
        MessagingException, AddressException, IOException,
        NoSuchProviderException {
        FetchProfile fp = new FetchProfile();
        fp.add(FetchProfile.Item.ENVELOPE);
        fp.add(FetchProfile.Item.FLAGS);
        fp.add("X-Mailer");
        folder.fetch(messages, fp);
        for (int i = 0; i < messages.length; i++) {

            // INBOX 폴더의 메세지를 사용자로부터 입력받은 곳(`listAddress`), roster 에 설정된 곳으로 전송
            Message message = messages[i];
            if (message.getFlags().contains(Flags.Flag.DELETED))
                continue;
            System.out.println("message received: " + message.getSubject());
            if (!roster.containsOneOf(message.getFrom()))
                continue;
            MimeMessage forward = new MimeMessage(session);
            InternetAddress result = null;
            Address[] fromAddress = message.getFrom();
            if (fromAddress != null && fromAddress.length > 0)
                result = new InternetAdress(fromAddress[0].toString());
            InternetAddress from = result;
            forward.setFrom(from);
            forward.setReplyTo(new Address[] {
                new InternetAddress(listAddress)});
            forward.addRecipients(Message.RecipientType.IO,
                listAddress);
            forward.addRecipients(Message.RecipientType.BCC,
                roster.getAddresses());
            String subject = message.getSubject();
            if (-1 == message.getSubject().indexOf(SUBJECT_MARKER)) 
                subject = SUBJECT_MARKER + " " + message.getSubject();
            forward.setSubject(subject);
            forward.setSentDate(message.getSentDate());
            forward.addHeader(LOOP_HEADER, listAddress);
            Object content = message.getContent();
            if (content instanceof Mulitpart)
                forward.setContent((Multipart) content);
            else
                forward.setText((String) content);
            Properties props = new Properties();
            props.put("mail.smtp.host", host.smtpHost);
            Session smtpSession = Session.getDefaultInstance(propes, null);
            Transport transport = smtpSession.getTransport("smtp");
            transport.connect(host.smtpHost,
                host.smtpUser, host.smtpPassword);
            transport.sendMessage(forward, roster.getAddresses());
            message.setFlag(Flags.Floag.DELETED, true);
        }
    }
}
```

자 파악이 끝났으면, 이걸 레거시 코드를 모르는 사람에게 핵심적인 부분만 알려준다고 가정하고, 어떻게 설명하면 좋을까요? 아마 최대한 간단하고 심플하게 핵심만 전달하려고 할거에요. 이런식으로요.

**"이 코드 특정 메일 서버로부터 주기적으로 확인하고 수신 메일이 있으면, 이를 읽어와 저희가 지정한 곳으로 전달하는 코드에요."**

이 내용이 코드에 담겨야해요. 그렇게 하기위해서는, 이 설명 안에서 책임들을 나열해봐야해요.

1. 주기적으로 서버에서 새로운 수신 메일을 확인한다. (주기적으로 확인하는 책임)
2. 새로운 수신 메일을 읽어온다. (메일을 읽어오는 책임)
4. 읽어온 메일을 다른 곳으로 전송한다. (메일을 전송하는 책임)

그리고 이 책임을 하나씩 클래스로 분배해줘요. 주기적으로 확인하는 책임은 `ListDriver`로, 메일을 읽어오는 책임은 `MailReceiver`로, 메일을 전송하는 책임은 `MessageForwarder`로 나눠줘요. 그리고 코딩을 시작해요. `listDriver` 에서 `run` 함수를 실행하면 주기적으로 `mailReceiver` 객체에서 `checkForMail` 함수로 새로운 수신메일이 있는지 확인하고 있으면 `MessageForwarder` 객체에 `forwardMessages` 함수를 호출하는 식으로 블라블라... 대략 아래처럼 작성할 수 있을거게요.

```java
class ListDriver {
    private final int interval;
    private final MailReceiver mailReceiver;

    ListDriver(int interval, MailReceiver mailReceiver) {
        this.interval = interval;
        this.mailReceiver = mailReceiver;
    }

    public void run() {
        while (true) {
            checkForMail();
            sleep();
        }
    }

    private void sleep() {
        try { 
            Thread.sleep(interval * 1000);
        } catch (InterruptedException e) {
        }
    }

    private void checkForMail() {
        try {
            mailReceiver.checkForMail();
        } catch (Exception e) {
            System.err.println(e);
            System.err.println("(retrying mail check)");
        }
        System.err.print(".");
    }
}

class MailReceiver {
    private final HostInformation host;
    private final MessageForwarder messageForwarder;

    MailReceiver(HostInformation host, MessageForwarder messageForwarder) {
        this.host = host;
        this.messageForwarder = messageForwarder;
    }

    public void checkForMail() {
        Properties properties = System.getProperties();
        Session session = Session.getDefaultInstance(
                properties, null);
        Store store = session.getStore("pop3");
        store.connect(host.pop3Host, -1,
                      host.pop3User, host.pop3Password);
        Folder defaultFolder = store.getDefaultFolder();
        if (defaultFolder == null) {
            System.err.prinln("Unable to open default folder");
            return;
        }
        Folder folder = defaultFolder.getFolder("INBOX");
        if (folder == null) {
            System.err.println("Unable to get: " + defaultFolder);
            return;
        }
        folder.open(Folder.READ_WRITE);

        try {
            if (folder.getMessageCount() != 0) {
                Message[] messages = folder.getMessages();
                FetchProfile fp = new FetchProfile();
                fp.add(FetchProfile.Item.ENVELOPE);
                fp.add(FetchProfile.Item.FLAGS);
                fp.add("X-Mailer");
                folder.fetch(messages, fp);
                messageForwarder.processMessages(messages);
            }
        } catch (Exception e) {
            System.err.println("message handling error");
            e.printStackTrace(System.err);
        } finally {
            folder.close(true);
            store.close();
        }
    }
}

class MessageForwarder {
    private final String listAddress;
    private final Roster roster;

    MessageForwarder(String listAddress, Roster roster) {
        this.listAddress = listAddress;
        this.roster = roster;
    }

    @Override
    public void processMessages(Message[] messages) {
        for (int i = 0; i < messages.length; i++) {
            Message message = messages[i];
            Message forward = create(message);
            if (forward == null) {
                continue;
            }

            Properties props = new Properties();
            props.put("mail.smtp.host", host.smtpHost);
            Session smtpSession = Session.getDefaultInstance(propes, null);
            Transport transport = smtpSession.getTransport("smtp");
            transport.connect(host.smtpHost,
                              host.smtpUser, host.smtpPassword);
            transport.sendMessage(forward, roster.getAddresses());

            message.setFlag(Flags.Floag.DELETED, true);
        }
    }

    private Message create(Message message) {
        if (message.getFlags().contains(Flags.Flag.DELETED)) { return null; }
        System.out.println("message received: " + message.getSubject());
        if (!roster.containsOneOf(message.getFrom())) { return null; }

        MimeMessage forward = new MimeMessage(session);
        InternetAddress result = null;
        Address[] fromAddress = message.getFrom();
        if (fromAddress != null && fromAddress.length > 0) {
            result = new InternetAdress(fromAddress[0].toString());
        }

        InternetAddress from = result;
        forward.setFrom(from);
        forward.setReplyTo(new Address[] {
                new InternetAddress(listAddress)
        });
        forward.addRecipients(Message.RecipientType.IO,
                              listAddress);
        forward.addRecipients(Message.RecipientType.BCC,
                              roster.getAddresses());

        String subject = message.getSubject();
        if (-1 == message.getSubject().indexOf(SUBJECT_MARKER)) {
            subject = SUBJECT_MARKER + " " + message.getSubject();
        }
        forward.setSubject(subject);
        forward.setSentDate(message.getSentDate());
        forward.addHeader(LOOP_HEADER, listAddress);

        Object content = message.getContent();
        if (content instanceof Mulitpart) { 
            forward.setContent((Multipart) content); 
        } else {
            forward.setText((String) content);
        }
        return forward;
    }
}
```

나름 깔끔해진 것 같은데, 헌데 코딩하다보면 **약간 이상한 점**을 찾을 수 있어요. `MailReceiver`와 `MessageForwarder` 둘의 관계가 조금 어색하다는 생각이 들어요. 저희가 머릿속으로 새로운 수신 메세지가 있으면 특정 곳으로 전달해야지 라는 생각을 가지고 있어서 어색하게 느껴지지 않을 수 있어요. 그런데, 지금 이 기능을 다 잊고 라이브러리로 `MailReceiver`가 있다고 가정해보아요. `MailReceiver`가 이미 `MessageForwarder`를 받아서 어떤 처리를 한다...? **라이브러리 사용자라** 새로운 메일이 있으면 어딘가로 메일을 전달하는 것이 아닌, 로컬에 저장하는 등 다른 일을 하고 싶을 수도 있을거 같아요.

그래서 중간에 인터페이스를 넣어서 좀더 **추상화/일반화**를 할 수 있을거 같아요. 인터페이스 이름은 `MessageProcessor`로 `processMessages(Messages[] messages)` 함수만 가져요. `MailReceiver`가 `MessageProcessor`를 사용하도록 해요. `MailForwarder`는 `MessageProcessor`를 구현하도록 해요. 그렇다면 이런 식으로 코딩될거에요.

```java
class MailReceiver {
    private final HostInformation host;
    private final MessageProcessor messageProcessor;

    MailReceiver(HostInformation host, MessageProcessor messageProcessor) {
        this.host = host;
        this.messageProcessor = messageProcessor;
    }

    public void checkForMail() {
        Properties properties = System.getProperties();
        Session session = Session.getDefaultInstance(
                properties, null);
        Store store = session.getStore("pop3");
        store.connect(host.pop3Host, -1,
                      host.pop3User, host.pop3Password);
        Folder defaultFolder = store.getDefaultFolder();
        if (defaultFolder == null) {
            System.err.prinln("Unable to open default folder");
            return;
        }
        Folder folder = defaultFolder.getFolder("INBOX");
        if (folder == null) {
            System.err.println("Unable to get: " + defaultFolder);
            return;
        }
        folder.open(Folder.READ_WRITE);

        try {
            if (folder.getMessageCount() != 0) {
                Message[] messages = folder.getMessages();
                FetchProfile fp = new FetchProfile();
                fp.add(FetchProfile.Item.ENVELOPE);
                fp.add(FetchProfile.Item.FLAGS);
                fp.add("X-Mailer");
                folder.fetch(messages, fp);
                messageProcessor.processMessages(messages);
            }
        } catch (Exception e) {
            System.err.println("message handling error");
            e.printStackTrace(System.err);
        } finally {
            folder.close(true);
            store.close();
        }
    }
}

interface MessageProcessor {
    void processMessages(Message[] messages);
}

class MessageForwarder implements MessageProcessor {
    private final String listAddress;
    private final Roster roster;

    MessageForwarder(String listAddress, Roster roster) {
        this.listAddress = listAddress;
        this.roster = roster;
    }

    @Override
    public void processMessages(Message[] messages) {
        for (int i = 0; i < messages.length; i++) {
            Message message = messages[i];
            Message forward = create(message);
            if (forward == null) {
                continue;
            }

            Properties props = new Properties();
            props.put("mail.smtp.host", host.smtpHost);
            Session smtpSession = Session.getDefaultInstance(propes, null);
            Transport transport = smtpSession.getTransport("smtp");
            transport.connect(host.smtpHost,
                              host.smtpUser, host.smtpPassword);
            transport.sendMessage(forward, roster.getAddresses());

            message.setFlag(Flags.Floag.DELETED, true);
        }
    }

    private Message create(Message message) {
        if (message.getFlags().contains(Flags.Flag.DELETED)) { return null; }
        System.out.println("message received: " + message.getSubject());
        if (!roster.containsOneOf(message.getFrom())) { return null; }

        MimeMessage forward = new MimeMessage(session);
        InternetAddress result = null;
        Address[] fromAddress = message.getFrom();
        if (fromAddress != null && fromAddress.length > 0) {
            result = new InternetAdress(fromAddress[0].toString());
        }

        InternetAddress from = result;
        forward.setFrom(from);
        forward.setReplyTo(new Address[] {
                new InternetAddress(listAddress)
        });
        forward.addRecipients(Message.RecipientType.IO,
                              listAddress);
        forward.addRecipients(Message.RecipientType.BCC,
                              roster.getAddresses());

        String subject = message.getSubject();
        if (-1 == message.getSubject().indexOf(SUBJECT_MARKER)) {
            subject = SUBJECT_MARKER + " " + message.getSubject();
        }
        forward.setSubject(subject);
        forward.setSentDate(message.getSentDate());
        forward.addHeader(LOOP_HEADER, listAddress);

        Object content = message.getContent();
        if (content instanceof Mulitpart) { 
            forward.setContent((Multipart) content); 
        } else {
            forward.setText((String) content);
        }
        return forward;
    }
}
```

이제 좀더 자연스러워졌죠. 그리고 한가지 더 발견할 수 있는데요. `MessageForwarder`의 테스트를 작성하면서 깨달았어요. 외부 시스템으로 이메일을 전송하 코드가 클래스 내에 명시되어 있어서 실제 환경을 만들지 않는 이상 테스트가 힘들다는 점이에요. 테스트를 위해서 `MailService` 인터페이스를 하나 넣어서 해결하려고 해요. (`MailService` 인터페이스는 `sendMessage(message)` 함수 하나만 가져요.) 실제 메일을 전송하는 코드는 `MailSender`라는 클래스로 `MailService`를 구현하여 작성도록 해요.

```java
class MessageForwarder implements MessageProcessor {
    private final String listAddress;
    private final Roster roster;
    private final MailService mailService;

    MessageForwarder(String listAddress, Roster roster, MailService mailService) {
        this.listAddress = listAddress;
        this.roster = roster;
        this.mailService = mailService;
    }

    @Override
    public void processMessages(Message[] messages) {
        for (int i = 0; i < messages.length; i++) {
            Message message = messages[i];
            Message forward = create(message);
            if (forward == null) {
                continue;
            }

            mailService.sendMessage(forward);
            message.setFlag(Flags.Floag.DELETED, true);
        }
    }

    private Message create(Message message) {
        if (message.getFlags().contains(Flags.Flag.DELETED)) { return null; }
        System.out.println("message received: " + message.getSubject());
        if (!roster.containsOneOf(message.getFrom())) { return null; }

        MimeMessage forward = new MimeMessage(session);
        InternetAddress result = null;
        Address[] fromAddress = message.getFrom();
        if (fromAddress != null && fromAddress.length > 0) {
            result = new InternetAdress(fromAddress[0].toString());
        }

        InternetAddress from = result;
        forward.setFrom(from);
        forward.setReplyTo(new Address[] {
                new InternetAddress(listAddress)
        });
        forward.addRecipients(Message.RecipientType.IO,
                              listAddress);
        forward.addRecipients(Message.RecipientType.BCC,
                              roster.getAddresses());

        String subject = message.getSubject();
        if (-1 == message.getSubject().indexOf(SUBJECT_MARKER)) {
            subject = SUBJECT_MARKER + " " + message.getSubject();
        }
        forward.setSubject(subject);
        forward.setSentDate(message.getSentDate());
        forward.addHeader(LOOP_HEADER, listAddress);

        Object content = message.getContent();
        if (content instanceof Mulitpart) { 
            forward.setContent((Multipart) content); 
        } else {
            forward.setText((String) content);
        }
        return forward;
    }
}

interface MailService {
    void sendMessage(Message message);
}

class MailSender implements MailService {
    private final HostInformation host;
    private final Roster roster;

    MailSender(HostInformation host, Roster roster){
        this.host = host;
        this.roster = roster;
    }

    @Override
    public void sendMessage(Message message) {
        Properties props = new Properties();
        props.put("mail.smtp.host", host.smtpHost);
        Session smtpSession = Session.getDefaultInstance(propes, null);
        Transport transport = smtpSession.getTransport("smtp");
        transport.connect(host.smtpHost,
                          host.smtpUser, host.smtpPassword);
        transport.sendMessage(forward, roster.getAddresses());
    }
}
```

이미 이렇게하면 `MailForwarder` 테스트 코드를 작성할 때, 가짜 `MailSender` 객체를 만들어서 넣어줄 수 있어요. 실제로 `sendMessage` 함수가 호출되었는지 어떤 인자가 들어왔는지 감시가 가능해져요. 손쉽게 테스트 코드를 짤 수 있게 되었어요. 예를 들면, 이렇죠.

```java
class MailForwarderTest(){
    @Test
    public void testProcessMessages() {
        MailSender mailSender = mock(MailSender.class);
        MailForwarder mailForwarder = new MailForwarder(mailSender);
        verfiy(mailSender).sendMessage(any());
    }
}
```

이렇게 해서 최종 코드는 이런 모습이 될거에요. 조금 깔끔해졌죠? (물론 이 예시로 해서 그렇긴한데, 여전히 덩치가 큰 함수들이 있어요. 앞에서 사용했던 방법들로 반복하면 작은 클래스, 작은 함수들로 더 깔끔하게 만들 수 있어요! 도전?) 레거시 코드를 리팩토링하길 바라시는 분들에게 조금 도움을 됐으면 해요. 그럼 이상입니다.

- UML
![](https://gyazo.com/793b1a805532a1689b02a9a18951efdd.png)

- 코드 (예제 코드라 실제로 돌아가진 않아요ㅜ)

```java
import java.io.IOException;
import java.util.Properties;

import javax.mail.*;
import javax.mail.internet.*;

public class MailingListServer {
    public static final String SUBJECT_MARKER = "[list]";
    public static final String LOOP_HEADER = "X-Loop";

    public static void main(String[] args) {
        if (args.length != 8) {
            System.err.println("Usage: java MailingList <popHost> " +
                               "<smtpHost> <pop3user> <pop3password> " +
                               "<smtpuser> <smtppassword> <listAddress> " +
                               "<replayinterval>");
            return;
        }
        HostInformation host = new HostInformation(
                args[0], args[1], args[2], args[3],
                args[4], args[5]);
        String listAddress = args[6];
        int interval = new Integer(args[7]).intValue();
        Roster roster = null;

        try {
            roster = new FileRoster("roster.txt");
        } catch (Excetion e) {
            System.err.println("unable to open roster.txt");
            return;
        }
        try {
            ListDriver listDriver = new ListDriver(interval, 
                new MailReceiver(host, 
                    new MessageForwarder(listAddress, roster,
                        new MailSender(host, roster))));
            listDriver.run();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    class ListDriver {
        private final int interval;
        private final MailReceiver mailReceiver;

        ListDriver(int interval, MailReceiver mailReceiver) {
            this.interval = interval;
            this.mailReceiver = mailReceiver;
        }

        public void run() {
            while (true) {
                checkForMail();
                sleep();
            }
        }

        private void sleep() {
            try { 
                Thread.sleep(interval * 1000);
            } catch (InterruptedException e) {
            }
        }

        private void checkForMail() {
            try {
                mailReceiver.checkForMail();
            } catch (Exception e) {
                System.err.println(e);
                System.err.println("(retrying mail check)");
            }
            System.err.print(".");
        }
    }

    class MailReceiver {
        private final HostInformation host;
        private final MessageProcessor messageProcessor;

        MailReceiver(HostInformation host, MessageProcessor messageProcessor) {
            this.host = host;
            this.messageProcessor = messageProcessor;
        }

        public void checkForMail() {
            Properties properties = System.getProperties();
            Session session = Session.getDefaultInstance(
                    properties, null);
            Store store = session.getStore("pop3");
            store.connect(host.pop3Host, -1,
                          host.pop3User, host.pop3Password);
            Folder defaultFolder = store.getDefaultFolder();
            if (defaultFolder == null) {
                System.err.prinln("Unable to open default folder");
                return;
            }
            Folder folder = defaultFolder.getFolder("INBOX");
            if (folder == null) {
                System.err.println("Unable to get: " + defaultFolder);
                return;
            }
            folder.open(Folder.READ_WRITE);

            try {
                if (folder.getMessageCount() != 0) {
                    Message[] messages = folder.getMessages();
                    FetchProfile fp = new FetchProfile();
                    fp.add(FetchProfile.Item.ENVELOPE);
                    fp.add(FetchProfile.Item.FLAGS);
                    fp.add("X-Mailer");
                    folder.fetch(messages, fp);
                    messageProcessor.processMessages(messages);
                }
            } catch (Exception e) {
                System.err.println("message handling error");
                e.printStackTrace(System.err);
            } finally {
                folder.close(true);
                store.close();
            }
        }
    }

    interface MessageProcessor {
        void processMessages(Message[] messages);
    }

    class MessageForwarder implements MessageProcessor {
        private final String listAddress;
        private final Roster roster;
        private final MailService mailService;

        MessageForwarder(String listAddress, Roster roster, MailService mailService) {
            this.listAddress = listAddress;
            this.roster = roster;
            this.mailService = mailService;
        }

        @Override
        public void processMessages(Message[] messages) {
            for (int i = 0; i < messages.length; i++) {
                Message message = messages[i];
                Message forward = create(message);
                if (forward == null) {
                    continue;
                }

                mailService.sendMessage(forward);
                message.setFlag(Flags.Floag.DELETED, true);
            }
        }

        private Message create(Message message) {
            if (message.getFlags().contains(Flags.Flag.DELETED)) { return null; }
            System.out.println("message received: " + message.getSubject());
            if (!roster.containsOneOf(message.getFrom())) { return null; }

            MimeMessage forward = new MimeMessage(session);
            InternetAddress result = null;
            Address[] fromAddress = message.getFrom();
            if (fromAddress != null && fromAddress.length > 0) {
                result = new InternetAdress(fromAddress[0].toString());
            }

            InternetAddress from = result;
            forward.setFrom(from);
            forward.setReplyTo(new Address[] {
                    new InternetAddress(listAddress)
            });
            forward.addRecipients(Message.RecipientType.IO,
                                  listAddress);
            forward.addRecipients(Message.RecipientType.BCC,
                                  roster.getAddresses());

            String subject = message.getSubject();
            if (-1 == message.getSubject().indexOf(SUBJECT_MARKER)) {
                subject = SUBJECT_MARKER + " " + message.getSubject();
            }
            forward.setSubject(subject);
            forward.setSentDate(message.getSentDate());
            forward.addHeader(LOOP_HEADER, listAddress);

            Object content = message.getContent();
            if (content instanceof Mulitpart) { 
                forward.setContent((Multipart) content); 
            } else {
                forward.setText((String) content);
            }
            return forward;
        }
    }

    interface MailService {
        void sendMessage(Message message);
    }

    class MailSender implements MailService {
        private final HostInformation host;
        private final Roster roster;

        MailSender(HostInformation host, Roster roster){
            this.host = host;
            this.roster = roster;
        }

        @Override
        public void sendMessage(Message message) {
            Properties props = new Properties();
            props.put("mail.smtp.host", host.smtpHost);
            Session smtpSession = Session.getDefaultInstance(propes, null);
            Transport transport = smtpSession.getTransport("smtp");
            transport.connect(host.smtpHost,
                              host.smtpUser, host.smtpPassword);
            transport.sendMessage(forward, roster.getAddresses());
        }
    }
}
```

#### 참고
- https://book.naver.com/bookdb/book_detail.nhn?bid=14032002