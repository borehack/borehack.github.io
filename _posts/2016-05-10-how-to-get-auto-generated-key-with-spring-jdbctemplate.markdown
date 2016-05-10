---
layout: post
title:  "How to get auto generated key with Spring jdbcTemplate"
date:   2016-05-10 09:02:00 +0000
---

If you are using [Spring Jdbc](https://spring.io/guides/gs/relational-data-access/) often you need the auto generated key from a database.
Sure you can just do `SELECT last_insert_id()` within the same transaction (`@Transactional`) for [MySQL](http://dev.mysql.com/doc/refman/5.7/en/information-functions.html#function_last-insert-id) and other database have similar functions. 
But if you change the database for testing or other reason you have to rewrite the code.
Therefore Spring has a generic solution, but most example do not use generic argument setter. See below how you would do it in a generic way.

{% highlight java %}
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.sql.Statement;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.ArgumentPreparedStatementSetter;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.PreparedStatementCreator;
import org.springframework.jdbc.support.GeneratedKeyHolder;
import org.springframework.jdbc.support.KeyHolder;
import org.springframework.stereotype.Component;

@Component
public class SpringDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public long insertWithId(final String sql, final Object... params) {
        final PreparedStatementCreator psc = new PreparedStatementCreator() {
            @Override
            public PreparedStatement createPreparedStatement(final Connection connection) throws SQLException {
                final PreparedStatement ps = connection.prepareStatement(sql,
                        Statement.RETURN_GENERATED_KEYS);
                if (params != null && params.length > 0) {
                    new ArgumentPreparedStatementSetter(params).setValues(ps);
                }
                return ps;
            }
        };

        final KeyHolder holder = new GeneratedKeyHolder();

        jdbcTemplate.update(psc, holder);

        return holder.getKey().longValue();
    }
}
{% endhighlight %}
