# Intent Layers

Intent Layers is an helper class to activate and deactivate intents, giving state to a mycroft skill

Each layer has different intents available to be executed, this allows for much easier sequential event programming


# Konami Code Example - layers with a single intent

        from mycroft.skills.intent_service import IntentLayers

        in initialize

        layers = [["KonamiUpIntent"], ["KonamiUpIntent"], ["KonamiDownIntent"], ["KonamiDownIntent"],
                ["KonamiLeftIntent"], ["KonamiRightIntent"], ["KonamiLeftIntent"], ["KonamiRightIntent"],
                ["KonamiBIntent"], ["KonamiAIntent"]]

        # 60 is the number of seconds for the layer timer
        self.layers = IntentLayers(self.emitter, layers, 60)

        to activate next layer/state -> self.layers.next()
        to activate previous layer/state -> self.layers.previous()
        to activate layer/state 0 -> self.layers.reset()
        to get current layer/state -> state = self.layers.current_layer

        each state/layer has a timer, after changing state this timer starts and at the end resets the layers
        to disable timer, after doing next/previous -> self.layers.stop_timer()

        to go directly to a layer do -> self.layers.activate_layer(layer_num)
        in this case no timer is started so you should also do - > self.layers.start_timer()

        on converse -> parse intent/utterance and manipulate layers if needed (bad sequence)

# Multiple Intent Layers

"Trees" can be made by making a IntentLayer for each intent, we can use layers as branches and do

        self.branch = IntentLayer(self.emitter, layers, 60)
        self.branch.disable()

and to activate later when needed

        self.branch.reset()

intent parsing in converse method can manipulate correct tree branch if needed

this allows for much more complex skills with each intent triggering their own sub-intents / sub - trees

on demand manipulation of branch layers may also open several unforeseen opportunities for intent data structures

        self.branch.add_layer([intent_list])
        self.branch.remove_layer(layer_num)
        self.branch.replace_layer(self, layer_num, intent_list)
        list_of_layers_with_this_intent_on_it = self.branch.find_layer(self, intent_list)
