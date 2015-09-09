FullStack Hello World
-----------------------------

http://rusthon-lang.blogspot.com/2015/09/fullstack-nosql-on-sql.html

Dataset
--------
Dataset is a database abstraction layer that makes using SQL dead simple.

https://dataset.readthedocs.org/en/latest/

```
git clone git://github.com/pudo/dataset.git
cd dataset/
sudo python setup.py install
```

Tornado Server
-------------
the tornado webserver saves objects into the database,
it gets from the client over the websocket.


```
./rusthon.py ./examples/hello_fullstack.md --run=myserver.py
```

@myserver.py
```python
#!/usr/bin/python


import dataset
import tornado
import tornado.ioloop
import tornado.web
import tornado.websocket
import os, sys, subprocess, datetime, json, time, mimetypes

PORT = int(os.environ.get("PORT", 8000))


## connect to database ##
##db = dataset.connect('sqlite:///mydatabase.db')
db = dataset.connect('sqlite:///:memory:')

# dataset will get or create `mytable`
table = db['mytable']


class MainHandler( tornado.web.RequestHandler ):

	def get(self, path=None):
		print('path', path)
		guess = path
		if not path or path == '/': guess = 'index.html'
		mime_type, encoding = mimetypes.guess_type(guess)
		if mime_type: self.set_header("Content-Type", mime_type)

		if path == 'favicon.ico' or path.endswith('.map'):
			self.write('')
		elif path and '.' in path:
			self.write( open(path).read() )
		else:
			self.write( open('index.html').read() )


class WebSocketHandler(tornado.websocket.WebSocketHandler):

	def open(self):
		print( 'websocket open' )
		print( self.request.connection )

	def on_message(self, msg):
		ob = json.loads( msg )
		## if a simple test string ##
		if isinstance(ob, str):
			print ob
		elif isinstance(ob, list):
			print 'doing database search'
			results = []
			for search in ob:
				assert isinstance(search, dict)
				## `**search` unpacks to something like `name="foo"`
				r = table.find_one( **search )
				if r: results.append(r)
			if results:
				print results
				self.write_message(json.dumps(results))
			else:
				self.write_message(json.dumps("nothing found in databases"))

		elif isinstance(ob, dict):
			print 'saving object into database'
			print ob
			table.insert( ob )
			self.write_message( json.dumps('updated database:'+str(len(table))) )


	def on_close(self):
		print('websocket closed')
		if self.ws_connection:
			self.close()



## Tornado Handlers ##
Handlers = [
	(r'/websocket', WebSocketHandler),
	(r'/(.*)', MainHandler),  ## order is important, this comes last.
]

print('<starting tornado server>')
app = tornado.web.Application(
	Handlers,
	#cookie_secret = 'some random text',
	#login_url = '/login',
	#xsrf_cookies = False,
)
app.listen( PORT )
tornado.ioloop.IOLoop.instance().start()


```

Client Side
-----------


@myapp
```rusthon
#backend:javascript
from runtime import *

ws = None

def on_open_ws():
	print 'websocket open'
	ws.send(JSON.stringify('hello server'))

def on_close_ws():
	print 'websocket close'

def on_message_ws(event):
	print 'on message', event
	msg = None
	if instanceof(event.data, ArrayBuffer):
		print 'got binary bytes', event.data.byteLength
		arr = new(Uint8Array(event.data))
		txt = String.fromCharCode.apply(None, arr)
		msg = JSON.parse(txt)
	else:
		msg = JSON.parse(event.data)


	pre = document.getElementById('RESULTS')
	if isinstance(msg, list):
		for res in msg:
			s = JSON.stringify(res)
			pre.appendChild( document.createTextNode(s+'\n') )
	else:
		pre.appendChild( document.createTextNode(msg+'\n') )

	print msg


def connect_ws():
	global ws
	print location.host
	addr = 'ws://' + location.host + '/websocket'
	print 'websocket test connecting to:', addr
	ws = new( WebSocket(addr) )
	ws.binaryType = 'arraybuffer'
	ws.onmessage = on_message_ws
	ws.onopen = on_open_ws
	ws.onclose = on_close_ws
	print ws


@debugger
def main():
	## connect websocket
	try: connect_ws()
	except: print 'could not connect to websocket'

	with 𝔼 as "document.createElement(%s)":
		with 𝕋 as "document.createTextNode(%s)":

			con = document.getElementById('FORM')
			h = 𝔼('h3')
			h.appendChild(𝕋('update database:'))
			con.appendChild(h)
			keys = ('first name', 'last name', 'age')
			fields = {}
			for key in keys:
				input = 𝔼('input')
				input.setAttribute('type', 'text')
				fields[key] = input
				con.appendChild(𝕋(key))
				con.appendChild(input)
				con.appendChild(𝔼('br'))

			button = 𝔼('button')
			button.appendChild(𝕋('submit'))
			con.appendChild(button)
			@bind(button.onclick)
			def onclick():
				ob = {}
				for key in fields.keys():
					elt = fields[key]
					ob[key] = elt.value

				jsondata = JSON.stringify(ob)
				ws.send(jsondata)

				searchform = document.getElementById('SEARCH')
				if searchform is None:
					searchform = 𝔼('div')
					searchform.setAttribute('id', 'SEARCH')
					document.body.appendChild(searchform)
					h = 𝔼('h3')
					h.appendChild(𝕋('search database:'))
					searchform.appendChild( h )

					search_fields = {}
					for key in keys:
						input = 𝔼('input')
						input.setAttribute('type', 'text')
						search_fields[key] = input
						searchform.appendChild(𝕋(key))
						searchform.appendChild(input)
						searchform.appendChild(𝔼('br'))


					sbutton = 𝔼('button')
					sbutton.appendChild(𝕋('search'))
					searchform.appendChild(sbutton)
					@bind(sbutton.onclick)
					def onsearch():
						s = []
						for key in search_fields.keys():
							elt = search_fields[key]
							## note ES6 syntax for a computed key name `[key]` ##
							o = {
								[key] : elt.value
							}
							s.append( o )
						ws.send( JSON.stringify(s) )



```

HTML
-----

@index.html
```html
<html>
<head>
<@myapp>
</head>
<body onload="main()">
<div id="FORM"></div>
<pre id="RESULTS">
</pre>
</body>
</html>
```