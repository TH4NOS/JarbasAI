# service backend

a helper abstract class was implemented, this handles timeouts and receiving data from other skill

# example implemented services usage

        import sys
        from os.path import dirname
        sys.path.append(dirname(dirname(__file__)))

        from service_object_recognition import ObjectRecogService
        from service_image_recognition import ImageRecogService
        from service_vision import VisionService

        def handle_describe_what_do_you_see_intent(self, message):
            # get vision feed and haar-cascade processing
            self.speak("Testing open cv vision")
            vision = VisionService(self.emitter)
            data = vision.get_data()
            feed = vision.get_feed()
            faces = vision.get_faces()
            self.speak("feed_data: " + str(data))

            # get tensor flow object recog api objects
            self.speak('Testing tensorflow object recognition')
            objrecog = ObjectRecogService(self.emitter, timeout=30)
            result = objrecog.recognize_objects(feed)
            objects = result.get("objects", []) # list of all detected objects
            labels = result.get("labels", {}) # label and ocurrences of each object with score > 30%
            self.speak("object_recog: " + str(labels))

            # get bvlc googlenet top 5 classification labels
            self.speak('Testing bvlc googlenet image recognition')
            imgrecog = ImageRecogService(self.emitter, timeout=130)
            classifications = imgrecog.get_classification(feed, server=True)
            self.speak("classifications: " + str(classifications))

# implementating a service

        from mycroft.skills.jarbas_service import BackendService

        class VisionService(ServiceBackend):
            def __init__(self, emitter=None, timeout=25, waiting_messages=None, logger=None):
                super(VisionService, self).__init__(name="VisionService", emitter=emitter, timeout=timeout, waiting_messages=waiting_messages, logger=logger)

            def get_feed(self):
                self.send_request("vision.feed.request")
                self.wait("vision.feed.result")
                if self.result is None:
                    self.result = {}
                return self.result.get("file")

            def get_data(self):
                self.send_request("vision_request")
                self.wait("vision_result")
                return self.result

            def get_faces(self, file=None):
                self.send_request("vision.faces.request", {"file": file})
                self.wait("vision.faces.result")
                if self.result is None:
                    self.result = {}
                return self.result.get("faces", [])

# full class

        class ServiceBackend(object):
            """
                Base class for all service implementations.

                Args:
                    name: name of service (str)
                    emitter: eventemitter or websocket object
                    timeout: time in seconds to wait for response (int)
                    waiting_messages: end wait on any of these messages (list)

            """

            def __init__(self, name, emitter=None, timeout=5, waiting_messages=None, logger=None):
                self.initialize(name, emitter, timeout, waiting_messages, logger)

            def initialize(self, name, emitter, timeout, waiting_messages, logger):
                """
                   initialize emitter, register events, initialize internal variables
                """
                self.name = name
                self.emitter = emitter
                self.timeout = timeout
                self.result = None
                self.waiting = False
                self.waiting_for = "any"
                if logger is None:
                    self.logger = getLogger(self.name)
                else:
                    self.logger = logger
                if waiting_messages is None:
                    waiting_messages = []
                self.waiting_messages = waiting_messages
                for msg in waiting_messages:
                    self.emitter.on(msg, self.end_wait)

            def send_request(self, message_type, message_data=None, message_context=None):
                """
                  send message
                """
                if message_data is None:
                    message_data = {}
                if message_context is None:
                    message_context = {"source": self.name, "waiting_for": self.waiting_messages}
                self.emitter.emit(Message(message_type, message_data, message_context))

            def wait(self, waiting_for="any"):
                """
                    wait until result response or time_out
                    waiting_for: message that ends wait, by default use any of waiting_messages list
                    returns True if result received, False on timeout
                """
                self.waiting_for = waiting_for
                if self.waiting_for != "any" and self.waiting_for not in self.waiting_messages:
                    self.emitter.on(waiting_for, self.end_wait)
                    self.waiting_messages.append(waiting_for)
                self.waiting = True
                start = time()
                elapsed = 0
                while self.waiting and elapsed < self.timeout:
                    elapsed = time() - start
                    sleep(0.3)

                return not self.waiting

            def end_wait(self, message):
                """
                    Check if this is the message we were waiting for and save result
                """
                if self.waiting_for == "any" or message.type == self.waiting_for:
                    self.result = message.data
                    self.waiting = False

            def get_result(self):
                """
                    return last processed result
                """
                return self.process_result()

            def process_result(self):
                """
                 process and return only desired data
                 """
                return self.result