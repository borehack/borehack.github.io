---
layout: post
title:  "Spring cloud aws SES message id"
date:   2016-04-24 13:02:13 +0000
---

create class in package and 

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

create the bean like so

{% highlight java %}
    @Bean
    MySimpleMailWrapper mySimpleMailWrapper(AmazonSimpleEmailService service) {
        return new MySimpleMailWrapper(service);
    }
{% endhighlight %}

