import pydot
from io import BytesIO
from PIL import Image
from python_qt_binding.QtGui import QPixmap, QImage
from python_qt_binding.QtWidgets import QGraphicsScene, QGraphicsPixmapItem

class LifecycleDrawing:
    def __init__(self):
        self.scene = QGraphicsScene()

    def draw_state_machine(self, current_state=None, transition_state=None):
        self.scene.clear()
        dot = self.create_dot_graph(current_state, transition_state)
        png_str = dot.create_png()
        image = Image.open(BytesIO(png_str))
        image_qt = self.pil2pixmap(image)
        pixmap_item = QGraphicsPixmapItem(image_qt)
        self.scene.addItem(pixmap_item)
        return self.scene

    def create_dot_graph(self, current_state=None, transition_state=None):
        dot = pydot.Dot(
            graph_type='digraph',
            rankdir='TB',
            splines='ortho',
            nodesep='1.5',
            ranksep='1.2',
            concentrate='true'
        )

        # Define colors
        default_color = '#FFFFFF'
        transition_color = '#FFFF00'
        success_color = '#00FF00'
        failure_color = '#FF0000'

        # Primary states with light blue fill
        primary_states = {
            'unconfigured': '#ADD8E6',
            'inactive': '#ADD8E6',
            'active': '#ADD8E6',
            'finalized': '#ADD8E6'
        }

        transition_states = {
            'Configuring': ('#FFFFE0', 'do: onConfigure()'),
            'CleaningUp': ('#FFFFE0', 'do: onCleanup()'),
            'ShuttingDown': ('#FFFFE0', 'do: onShutdown()'),
            'Activating': ('#FFFFE0', 'do: onActivate()'),
            'Deactivating': ('#FFFFE0', 'do: onDeactivate()'),
            'ErrorProcessing': ('#FFFFE0', 'do: onError()')  # Keep normal fill here
        }

        # Add a solid start node (level 1)
        start_node = pydot.Node('start', shape='point', width=0.2, style='filled', fillcolor='black')
        dot.add_node(start_node)

        # Helper to add error cross overlay to label if needed
        def add_error_cross(label, add_cross):
            # Unicode heavy multiplication X = ✗ U+2717 (works well visually)
            return label + "  ✗" if add_cross else label

        # Level 1: unconfigured
        unconfigured_fill = primary_states['unconfigured']
        if 'unconfigured' == current_state:
            if transition_state == 'in-progress':
                unconfigured_fill = transition_color
            elif transition_state == 'success':
                unconfigured_fill = success_color
            elif transition_state == 'failed':
                unconfigured_fill = failure_color
        unconfigured_node = pydot.Node('unconfigured', style='filled', fillcolor=unconfigured_fill, shape='box')
        dot.add_node(unconfigured_node)

        dot.add_edge(pydot.Edge('start', 'unconfigured', style='solid'))

        # Level 2: ErrorProcessing, CleaningUp, Configuring
        level2 = pydot.Subgraph(rank='same', name='level2')
        for state in ['ErrorProcessing', 'CleaningUp', 'Configuring']:
            fillcolor, label = transition_states[state]
            add_cross = False
            if state == current_state:
                if transition_state == 'in-progress':
                    fillcolor = transition_color
                elif transition_state == 'success':
                    fillcolor = success_color
                elif transition_state == 'failed':
                    fillcolor = failure_color
                    # Add red cross on this node label to mark error happened here
                    add_cross = True
            node_label = add_error_cross(f"{state}\n{label}", add_cross)
            node = pydot.Node(state, style='filled', fillcolor=fillcolor, shape='box', label=node_label)
            level2.add_node(node)
        dot.add_subgraph(level2)

        # Add a small red dot node next to ErrorProcessing if it is the current error state (failed)
        if current_state == 'ErrorProcessing' and transition_state == 'failed':
            red_dot = pydot.Node(
                'error_dot',
                shape='circle',
                fixedsize='true',
                width=0.15,
                height=0.15,
                style='filled',
                fillcolor=failure_color,
                label='',  # no label
                penwidth=0,
            )
            dot.add_node(red_dot)
            # Connect red dot to ErrorProcessing with a thin visible edge to force proximity
            edge = pydot.Edge(
                'ErrorProcessing', 'error_dot',
                color=failure_color,
                style='dashed',
                constraint='false',
                arrowsize='0',
                penwidth='1'
            )
            dot.add_edge(edge)

        # Level 3
        level3 = pydot.Subgraph(rank='same', name='level3')
        for state in ['inactive', 'ShuttingDown']:
            fillcolor = primary_states.get(state, default_color)
            add_cross = False
            if state == current_state:
                if transition_state == 'in-progress':
                    fillcolor = transition_color
                elif transition_state == 'success':
                    fillcolor = success_color
                elif transition_state == 'failed':
                    fillcolor = failure_color
                    add_cross = True
            node_label = state
            if add_cross:
                node_label += "  ✗"
            node = pydot.Node(state, style='filled', fillcolor=fillcolor, shape='box', label=node_label)
            level3.add_node(node)
        dot.add_subgraph(level3)

        # Level 4
        level4 = pydot.Subgraph(rank='same', name='level4')
        for state in ['finalized', 'Deactivating', 'Activating']:
            if state in primary_states:
                fillcolor = primary_states[state]
                label = None
                shape = 'box'
            else:
                fillcolor, label = transition_states[state]
                shape = 'box'
            add_cross = False
            if state == current_state:
                if transition_state == 'in-progress':
                    fillcolor = transition_color
                elif transition_state == 'success':
                    fillcolor = success_color
                elif transition_state == 'failed':
                    fillcolor = failure_color
                    add_cross = True
            node = pydot.Node(state, style='filled', fillcolor=fillcolor, shape=shape)
            if label:
                label_text = f"{state}\n{label}"
                if add_cross:
                    label_text += "  ✗"
                node.set_label(label_text)
            else:
                # If no label property, add cross on node name
                if add_cross:
                    node.set_label(f"{state}  ✗")
                else:
                    node.set_label(state)
            level4.add_node(node)
        dot.add_subgraph(level4)

        # Level 5
        stop_node = pydot.Node('stop', shape='point', width=0.2, style='filled', fillcolor='black')
        dot.add_node(stop_node)

        level5 = pydot.Subgraph(rank='same', name='level5')
        fillcolor = primary_states['active']
        add_cross = False
        if 'active' == current_state:
            if transition_state == 'in-progress':
                fillcolor = transition_color
            elif transition_state == 'success':
                fillcolor = success_color
            elif transition_state == 'failed':
                fillcolor = failure_color
                add_cross = True
        active_node = pydot.Node('active', style='filled', fillcolor=fillcolor, shape='box')
        if add_cross:
            active_node.set_label("active  ✗")
        level5.add_node(stop_node)
        level5.add_node(active_node)
        dot.add_subgraph(level5)

        dot.add_edge(pydot.Edge('finalized', 'stop', style='solid'))

        edges = [
            ('unconfigured', 'Configuring', 'configure()', 'normal'),
            ('unconfigured', 'ShuttingDown', 'shutdown()', 'normal'),

            ('Configuring', 'inactive', 'onConfigure[SUCCESS]', 'normal'),
            ('Configuring', 'unconfigured', 'onConfigure[FAILURE]', 'normal'),

            ('inactive', 'Activating', 'activate()', 'normal'),
            ('inactive', 'CleaningUp', 'cleanup()', 'normal'),
            ('inactive', 'ShuttingDown', 'shutdown()', 'normal'),

            ('Activating', 'active', 'onActivate[SUCCESS]', 'normal'),
            ('Activating', 'inactive', 'onActivate[FAILURE]', 'normal'),

            ('active', 'Deactivating', 'deactivate()', 'normal'),
            ('active', 'ShuttingDown', 'shutdown()', 'normal'),

            ('Deactivating', 'inactive', 'onDeactivate[SUCCESS]', 'normal'),

            ('CleaningUp', 'unconfigured', 'onCleanup[SUCCESS]', 'normal'),

            ('ShuttingDown', 'finalized', 'onShutdown[SUCCESS]', 'normal'),

            ('ErrorProcessing', 'unconfigured', 'onError[SUCCESS]', 'normal'),
            ('ErrorProcessing', 'finalized', 'onError[FAILURE]', 'normal'),
        ]

        for from_state, to_state, label, style in edges:
            edge_kwargs = {
                'xlabel': label,
                'fontsize': '10',
                'fontname': 'Arial',
                'color': 'black',
                'style': 'solid',
                'label': '',          # keep edge label empty to avoid default overlap
                'labeldistance': '2',  # default is 1.0, increase to move label farther
                'labelangle': '45',   # angle in degrees, tweak to move label position
            }

            edge = pydot.Edge(from_state, to_state, **edge_kwargs)
            dot.add_edge(edge)


        return dot

    def pil2pixmap(self, image):
        image = image.convert("RGBA")
        data = image.tobytes("raw", "BGRA")
        qimage = QImage(data, image.width, image.height, QImage.Format_RGBA8888)
        pixmap = QPixmap.fromImage(qimage)
        return pixmap
