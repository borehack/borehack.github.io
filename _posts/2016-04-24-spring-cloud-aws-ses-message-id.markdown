---
layout: post
title:  "Spring cloud aws SES message id"
date:   2016-04-24 13:02:13 +0000
---

If you are using [Spring Cloud AWS](http://cloud.spring.io/spring-cloud-aws/) and want to send a eMail with Simple Email Service (SES),
you notice that you can not receive the message id provided by AWS.

If you need the message id to associate bounce or complaint notifications within your system that is kind of bad.

The class `SimpleEmailServiceJavaMailSender` implements `JavaMailSender` where all send methods are void. So they will not return the message id.
If you look into `SimpleEmailServiceJavaMailSender` you can see that they actually use the AWS SES SDK and they have access to the message id via the `SendRawEmailResult`.

Therefore we write a wrapper around `SimpleEmailServiceJavaMailSender` and later create this bean. The method ses is basically a copy of the send method in `SimpleEmailServiceJavaMailSender`.
But with one exception it will return the message id as a string.

{% highlight java %}
package org.springframework.cloud.aws.mail.simplemail;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.util.HashMap;
import java.util.Map;

import javax.mail.MessagingException;
import javax.mail.internet.MimeMessage;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.mail.MailException;
import org.springframework.mail.MailParseException;
import org.springframework.mail.MailPreparationException;
import org.springframework.mail.MailSendException;
import org.springframework.stereotype.Component;

import com.amazonaws.services.simpleemail.AmazonSimpleEmailService;
import com.amazonaws.services.simpleemail.model.RawMessage;
import com.amazonaws.services.simpleemail.model.SendRawEmailRequest;
import com.amazonaws.services.simpleemail.model.SendRawEmailResult;

@Component
public class MySimpleMailWrapper extends SimpleEmailServiceJavaMailSender {
    private static final Logger LOGGER = LoggerFactory.getLogger(SimpleEmailServiceMailSender.class);

    public MySimpleMailWrapper(AmazonSimpleEmailService amazonSimpleEmailService) {
        super(amazonSimpleEmailService);
    }

    public String ses(MimeMessage mimeMessage) throws MailException {
        try {
            RawMessage rm = createRawMessage(mimeMessage);
            SendRawEmailResult sendRawEmailResult = getEmailService().sendRawEmail(new SendRawEmailRequest(rm));
            if (LOGGER.isDebugEnabled()) {
                LOGGER.debug("Message with id: {0} successfully send", sendRawEmailResult.getMessageId());
            }
            return sendRawEmailResult.getMessageId();
        } catch (Exception e) {
            // Ignore Exception because we are collecting and throwing all if any
            // noinspection ThrowableResultOfMethodCallIgnored

            Map<Object, Exception> failedMessages = new HashMap<>();
            failedMessages.put(mimeMessage, e);

            throw new MailSendException(failedMessages);
        }
    }

    private RawMessage createRawMessage(MimeMessage mimeMessage) {
        ByteArrayOutputStream out;
        try {
            out = new ByteArrayOutputStream();
            mimeMessage.writeTo(out);
        } catch (IOException e) {
            throw new MailPreparationException(e);
        } catch (MessagingException e) {
            throw new MailParseException(e);
        }
        return new RawMessage(ByteBuffer.wrap(out.toByteArray()));
    }
}
{% endhighlight %}

You have to create this bean like so in a Class associated with the @Configuration annotation.

{% highlight java %}
    @Bean
    MySimpleMailWrapper mySimpleMailWrapper(AmazonSimpleEmailService service) {
        return new MySimpleMailWrapper(service);
    }
{% endhighlight %}

## Configuration

This is an extra part on how to configure the messaging package of [spring cloud aws](https://github.com/spring-cloud/spring-cloud-aws).
To configure messaging we can use the auto configuration package from spring cloud aws.

{% highlight xml %}
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-aws-autoconfigure</artifactId>
    <version>1.0.4.RELEASE</version>
</dependency>
{% endhighlight %}

after you enable the auto configuration package you will see a line like this during startup. this will slow down the startup process.

{% highlight xml %}
12:49:16.020 [restartedMain] DEBUG c.a.internal.EC2MetadataClient - Connecting to EC2 instance metadata service at URL: http://169.254.169.254/latest/meta-data/instance-id
{% endhighlight %}

as long as you only need messaging and no E2 environment you want to disable the auto-configuration for the EC2 environment.

{% highlight java %}
@SpringBootApplication
@EnableAutoConfiguration(exclude = { ContextResourceLoaderAutoConfiguration.class,
        ContextResourceLoaderConfiguration.class, ContextInstanceDataAutoConfiguration.class })
public class Application {

    public static void main(String[] args) throws Throwable {
        SpringApplication.run(Application.class, args);
    }
}
{% endhighlight %}

you can disable more auto configuration if you want to. the following link describe how to disable auto configuration with spring boot.

https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-auto-configuration.html
