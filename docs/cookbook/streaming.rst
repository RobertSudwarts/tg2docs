Streaming Response
======================

Streaming permits to your controller to send data that yet has to be created to the client,
this can be really useful when your app needs to send an huge amount of data to the client,
much more data than you are able to keep in memory.

Streaming can also be useful when you have to send content to the client that yet has to be
generated when you provide an answer to the client keeping an open connection between the
application and the client itself.

In TurboGears2 streaming can be achieved returning a generator from your controllers.

Making your application streaming compliant
---------------------------------------------

So the first thing you want to do when using streaming is disabling any middleware
that edits the content of your response. Since version 2.3 disabling debug mode
is not required anymore.

Most middlewares like debugbar, ToscaWidgets and so on will avoid touching your
response if it is not of **text/html** content type.
So streaming files or json is usually safe.

Streaming with Generators
-----------------------------

Streaming involves returning a generator from your controller, this will let
the generator create the content while being read by client.

.. code-block:: python

    @expose(content_type='application/json')
    def stream_list(self):
        def output_pause():
            num = 0
            yield '['
            while num < 9:
                num += 1
                yield '%s, ' % num
                time.sleep(1)
            yield '10]'
        return output_pause()

This simple example will slowly stream the numbers from 1 to 10 in a json array.

Accessing TurboGears objects
------------------------------

While streaming content is quite simple some teardown functions get executed
before your streamer, this has the side effect of changing how your application
behaves.

Accessing Request
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since version 2.3 all the global turbogears objects are accessible
while running the generator. So if you need to have access to them,
you can freely read and write to them. Just keep in mind that your
response has already been started, so it is not possible to change
your outgoing response while running the generator.

.. code-block:: python

    @expose(content_type='text/css')
    def stream(self):
        def output_pause(req):
            num = 0
            while num < 10:
                num += 1
                yield '%s/%s\n' % (tg.request.path_info, num)
                time.sleep(1)
        return output_pause()

This example, while not returning any real css, shows how it is possible
to access the turbogears request inside the generator.

Reading from Database
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since version 2.3 is is possible to read from the database as usual:

.. code-block:: python

    @expose(content_type='application/json')
    def stream_db(self):
        def output_pause():
            num = 0
            yield '['
            while num < 9:
                u = DBSession.query(model.User).filter_by(user_id=num).first()
                num += 1
                yield u and '%d, ' % u.user_id or 'null, '
                time.sleep(1)
            yield 'null]'
        return output_pause()

Writing to Database
~~~~~~~~~~~~~~~~~~~~~~~~~

If you need to write data on the database you will have to manually flush the session
and commit the transaction. This is due to the fact that TurboGears2
won't be able to do it for you as the request flow already ended.

.. code-block:: python

    @expose(content_type='application/json')
    def stream_list(self):
        def output_pause():
            import transaction
            num = 0
            while num < 9:
                DBSession.add(model.Permission(permission_name='perm_%s'%num))
                yield 'Added Permission\n'
                num += 1
                time.sleep(1)
            DBSession.flush()
            transaction.commit()
        return output_pause()

