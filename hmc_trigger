import picamera
import socket
import struct
from threading import Thread, Event, Timer
from queue import Queue
import time
from subprocess import call
import os

HOST, PORT = ('192.168.1.184', 5005)


class VideoWriter(Thread):
	
	def __init__(self, vidfile, q):
		super(VideoWriter, self).__init__()
		
		PATH = '/home/pi/Documents/'
		FILE = vidfile.decode('utf-8')
		
		self.rawname = '{}{}.h264'.format(PATH, FILE)
		self.newname = '{}{}.mkv'.format(PATH, FILE)
		
		self.vidfile = open(self.rawname, 'wb')
		self.q = q
	
	def run(self):
		while True:
			try:
				d = self.q.get_nowait()
				if isinstance(d, int):
					print('Closing file')
					self.vidfile.close()
					break
				else:
					self.vidfile.write(d)
			except:
				pass
		
		# Set up FFMPEG command
		print('Converting RAW to MKV...')
		cmd = [
			'ffmpeg',
			'-r',
			'30',
			'-i',
			self.rawname,
			'-vsync',
			'1',
			'-c:v',
			'copy',
			'-an',
			'-copyts',
			'-start_at_zero',
			self.newname
			]
		call(cmd)
		print('Conversion complete')


class UDPListener(Thread):
	
	def __init__(self, HOST, PORT, rec_on_off, q):
		super(UDPListener, self).__init__()
		
		# Set up UDP Listeneing
		self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
		self.sock.setblocking(False)
		self.sock.bind((HOST, PORT))

		# Set up thread parameters
		self.rec_on_off = rec_on_off
		self.q = q
		self.stoprequest = Event()
		self.daemon = True

	def run(self):
		while not self.stoprequest.isSet():
			try:
				data, addr = self.sock.recvfrom(1024)
				print(data, addr)
				if len(data) > 0:
					print("Record triggered")
					self.rec_on_off(True)
					self.writer = VideoWriter(data, self.q)
					t = Timer(1.0, self.start_writer)
					t.start()
				else:
					print("Stop triggered")
					self.rec_on_off(False)
					self.q.put_nowait(0)
			except:
				pass

		self.sock.close()
	
	def start_writer(self):
		print('Writer started')
		self.writer.start()

	def join(self, timeout=None):
		self.stoprequest.set()
		super(UDPListener, self).join(timeout)


class FrameBuffer(object):
	
	def __init__(self):
		
		# Set up queue and writer
		self.vidqueue = Queue()
		self.rectrig = False
		self.writer = None
		self.frame = b''
		
		# Create UDP Object
		global HOST
		global PORT
		self.listener = UDPListener(HOST, PORT, self.rec_on_off, self.vidqueue)
		self.listener.start()
		print('Server started at {} port {}'.format(HOST, PORT))

	def write(self, buf):
		#if buf.startswith(b'\xff\xd8'):
		#	if self.frame != b'':
		if self.rectrig:
			self.vidqueue.put_nowait(buf)
		#	self.frame = buf
		#else:
		#	self.frame += buf
	
	def rec_on_off(self, toggle):
		self.rectrig = toggle
	
	def flush(self):
		self.listener.join()


def set_camera(camera):
	# Fix values
	camera.resolution = (640, 480)
	camera.framerate = 30
	camera.iso = 100
	time.sleep(2)
	camera.shutter_speed = camera.exposure_speed
	camera.exposure_mode = 'off'
	g = camera.awb_gains
	camera.awb_mode = 'off'
	camera.awb_gains = g


if __name__ == '__main__':
	#HOST, PORT = '192.168.1.184', 5005

	# Open the videostream
	camera = picamera.PiCamera()
	set_camera(camera)
	
	print("Warming up PiCamera...")
	camera.start_preview()
	time.sleep(2)
	print("PiCamera ready.")
	
	frame_buffer = FrameBuffer()
	
	camera.start_recording(frame_buffer,
						   format='h264',
						   profile='high',
						   inline_headers=True,
						   intra_period=1,
						   sei=False,
						   sps_timing=True,
						   bitrate=10000000,
						   quality=20)
	
	#camera.start_recording(frame_buffer,
	#					   format='mjpeg',
	#					   quality=50)
	
	try:
		while True:
			pass
	except (KeyboardInterrupt, SystemExit):
		camera.stop_preview()
