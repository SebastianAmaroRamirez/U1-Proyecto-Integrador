# U1-Proyecto-Integrador
---
## Escenario en Blender

El objetivo principal del código es la generación procedural de un entorno arquitectónico tridimensional —específicamente un pasillo con tramos rectos y curvos— eliminando la necesidad de modelado manual y permitiendo una iteración rápida de diseño.

En el flujo de trabajo de la industria 3D (cine, videojuegos o visualización arquitectónica), la creación de escenarios repetitivos o basados en patrones geométricos puede ser ineficiente. Este script aborda dicho desafío mediante tres pilares fundamentales:

- Geometría Matemática: El uso de funciones trigonométricas para posicionar elementos en el espacio con precisión absoluta.

- Lógica de Materiales Inteligente: Un sistema que asigna propiedades visuales y de emisión de luz basándose en el análisis de color.

- Cinematografía Automatizada: La implementación de un sistema de cámara basado en restricciones (constraints) que garantiza un recorrido fluido y profesional a través de una curva de trayectoria.
--------

:notebook: **Codigo en Blender:** 

````
import bpy
import math

def crear_material(nombre, color_rgb):
    mat = bpy.data.materials.new(name=nombre)
    mat.use_nodes = True
    bsdf = mat.node_tree.nodes["Principled BSDF"]
    bsdf.inputs['Base Color'].default_value = (*color_rgb, 1.0) 
    bsdf.inputs['Roughness'].default_value = 0.7 
    # Añadimos un poco de emisión para que el color resalte más
    if color_rgb[0] > 0.5 or color_rgb[2] > 0.5: # Si es un color de detalle
        bsdf.inputs['Emission Color'].default_value = (*color_rgb, 1.0)
        bsdf.inputs['Emission Strength'].default_value = 0.5
    return mat

def generar_escenario():
    # 1. Limpiar la escena previa
    bpy.ops.object.select_all(action='SELECT')
    bpy.ops.object.delete()

    # 2. Definir materiales (CAMBIO DE COLOR AQUÍ)
    mat_pared_a = crear_material("ParedOscura", (0.05, 0.05, 0.05))
    # He cambiado el naranja (0.8, 0.2, 0.0) por un Azul Eléctrico (0.0, 0.4, 1.0)
    mat_pared_b = crear_material("ParedDetalle", (0.0, 0.5, 0.0)) 

    # 3. Parámetros del escenario
    largo_pasillo = 10
    ancho_pasillo = 4
    radio_curva = 12 

    # 4a. Tramo Recto
    for i in range(largo_pasillo):
        bpy.ops.mesh.primitive_cube_add(location=(-ancho_pasillo, i * 2, 1))
        pared_izq = bpy.context.active_object
        if i % 2 == 0:
            pared_izq.data.materials.append(mat_pared_a)
        else:
            pared_izq.data.materials.append(mat_pared_b)
            pared_izq.scale.z = 1.5

        bpy.ops.mesh.primitive_cube_add(location=(ancho_pasillo, i * 2, 1))
        pared_der = bpy.context.active_object
        pared_der.data.materials.append(mat_pared_a)

    # 4b. Tramo Curvo
    cy = (largo_pasillo - 1) * 2 
    cx = radio_curva 

    for j in range(1, largo_pasillo + 1):
        angulo = math.pi - (j * (math.pi / 2) / largo_pasillo)
        rotacion_z = math.pi - angulo 

        # Pared Izquierda Curva
        x_izq = cx + (radio_curva + ancho_pasillo) * math.cos(angulo)
        y_izq = cy + (radio_curva + ancho_pasillo) * math.sin(angulo)
        bpy.ops.mesh.primitive_cube_add(location=(x_izq, y_izq, 1), rotation=(0, 0, rotacion_z))
        pared_izq_curva = bpy.context.active_object
        
        # Usamos i (último valor del loop anterior) + j para mantener la alternancia
        if (10 + j) % 2 == 0:
            pared_izq_curva.data.materials.append(mat_pared_a)
        else:
            pared_izq_curva.data.materials.append(mat_pared_b)
            pared_izq_curva.scale.z = 1.5

        # Pared Derecha Curva
        x_der = cx + (radio_curva - ancho_pasillo) * math.cos(angulo)
        y_der = cy + (radio_curva - ancho_pasillo) * math.sin(angulo)
        bpy.ops.mesh.primitive_cube_add(location=(x_der, y_der, 1), rotation=(0, 0, rotacion_z))
        pared_der_curva = bpy.context.active_object
        pared_der_curva.data.materials.append(mat_pared_a)

    # 5. Agregar suelo
    bpy.ops.mesh.primitive_plane_add(size=1, location=(radio_curva/2, cy/2 + radio_curva/2, 0))
    suelo = bpy.context.active_object
    suelo.scale.x = (ancho_pasillo * 2) + radio_curva + 10
    suelo.scale.y = (largo_pasillo * 2) + radio_curva + 10
    suelo.data.materials.append(crear_material("Suelo", (0.02, 0.02, 0.02)))

    # 6. Agregar Luces
    bpy.ops.object.light_add(type='SUN', location=(10, 10, 10), rotation=(math.radians(-45), math.radians(30), 0))
    sun = bpy.context.active_object
    sun.data.energy = 2 

    # 7. Agregar Cámara
    bpy.ops.object.camera_add(location=(0, 0, 0), rotation=(0, 0, 0))
    camera = bpy.context.active_object
    bpy.context.scene.camera = camera

    # --- 8. CREAR EL CAMINO (cam_path) ---
    curve_data = bpy.data.curves.new('CamPathData', type='CURVE')
    curve_data.dimensions = '3D'
    curve_data.use_path = True 
    spline = curve_data.splines.new('POLY') 
    
    puntos_camino = [(0, -6, 1.5), (0, cy, 1.5)]
    pasos_curva = 20
    for step in range(1, pasos_curva + 1):
        progreso = step / float(pasos_curva)
        angulo_c = math.pi - (progreso * (math.pi / 2))
        px = cx + radio_curva * math.cos(angulo_c)
        py = cy + radio_curva * math.sin(angulo_c)
        puntos_camino.append((px, py, 1.5))

    spline.points.add(len(puntos_camino) - 1)
    for idx, pt in enumerate(puntos_camino):
        spline.points[idx].co = (*pt, 1.0) 

    cam_path = bpy.data.objects.new('Cam_Path', curve_data)
    bpy.context.scene.collection.objects.link(cam_path)

    # --- 9. CONFIGURAR RESTRICCIONES ---
    bpy.ops.object.empty_add(type='PLAIN_AXES', location=(0, 0, 0))
    cam_target = bpy.context.active_object
    cam_target.name = "Cam_Target"

    fp_target = cam_target.constraints.new(type='FOLLOW_PATH')
    fp_target.target = cam_path
    fp_target.use_fixed_location = True

    follow_path = camera.constraints.new(type='FOLLOW_PATH')
    follow_path.target = cam_path
    follow_path.use_fixed_location = True 

    track_to = camera.constraints.new(type='TRACK_TO')
    track_to.target = cam_target
    track_to.track_axis = 'TRACK_NEGATIVE_Z'
    track_to.up_axis = 'UP_Y'

    # --- 10. ANIMAR ---
    bpy.context.scene.frame_start = 1
    bpy.context.scene.frame_end = 200
    follow_path.offset_factor = 0.0
    fp_target.offset_factor = 0.05 
    camera.keyframe_insert(data_path=f'constraints["{follow_path.name}"].offset_factor', frame=1)
    cam_target.keyframe_insert(data_path=f'constraints["{fp_target.name}"].offset_factor', frame=1)

    follow_path.offset_factor = 0.95 
    fp_target.offset_factor = 1.0 
    camera.keyframe_insert(data_path=f'constraints["{follow_path.name}"].offset_factor', frame=200)
    cam_target.keyframe_insert(data_path=f'constraints["{fp_target.name}"].offset_factor', frame=200)

    bpy.context.view_layer.update()

generar_escenario()

````
-----
:green_book:  Explicacion 
---

Este código de Python para Blender es un script completo que genera un pasillo procedural con un tramo recto y una curva, y luego configura una animación de cámara cinematográfica que recorre dicho pasillo.

### **1. Función** ``crear_material``

Esta función automatiza la creación de colores.

- Crea un nuevo material y activa los Nodos.

- Configura el color base y la rugosidad (0.7 para que no sea muy brillante).

- **Lógica de Emisión:** Si el color es brillante (como el azul eléctrico definido), le añade un efecto de "luz propia" (emisión) para que resalte en la oscuridad.

### **2. Limpieza y Configuración Inicial**

- ``bpy.ops.object.select_all`` y ``delete``: Borra todo lo que haya en tu escena de Blender para empezar de cero cada vez que ejecutas el script.

- Define dos materiales: uno Oscuro (fondo) y uno Azul Eléctrico (detalles).

### **3. Construcción del Pasillo (Tramo Recto)**

Usa un bucle  ``for `` para colocar cubos a ambos lados:

- **Pared Derecha:** Cubos oscuros fijos.

- **Pared Izquierda:** Alterna cada dos cubos. Si el índice es impar, el cubo es azul y más alto ( ``scale.z = 1.5 ``), creando un ritmo visual de "columnas de luz".

### **4. Construcción del Pasillo (Tramo Curvo)**

Aquí es donde entra la matemática (``math.sin`` y ``math.cos``):

- Calcula una trayectoria circular basada en el ``radio_curva``.

- Coloca cubos rotándolos gradualmente para que sigan la forma del arco.

- Mantiene la misma lógica de diseño: la pared exterior es oscura y la interior tiene detalles azules alternados.

### **5. Suelo, Luces y Cámara**

- **Suelo:** Añade un plano gigante de color casi negro.

- **Luz:** Crea un "Sol" (``SUN``) inclinado para dar sombras y profundidad.

- **Cámara:** Crea la cámara en el origen.

### **6. El Camino de la Cámara (``CamPath``)**

El script crea una **Curva (Path)** invisible que sigue la forma exacta del pasillo:

1. Define puntos en línea recta.

2. Calcula puntos en curva (usando 20 pasos para que sea suave).

3. Une los puntos en un objeto de curva llamado ``Cam_Path``.

### **7. Sistema de "Target" y Restricciones**

Esta es la parte técnica para que la cámara se vea profesional:

- Crea un **Empty** (un objeto invisible) llamado ``Cam_Target``.

- **Constraint Follow Path:** Obliga tanto a la cámara como al Empty a seguir la curva.

- **Constraint Track To:** Obliga a la cámara a mirar siempre hacia el Empty.

### **8. Animación (Keyframes)**

- Define que la animación dura **200 cuadros**.

- En el **Frame 1**: La cámara está al inicio de la curva (0%).

- En el **Frame 200**: La cámara llega al final (95-100%).

- Inserta los "keyframes" automáticamente para que, al dar al Play, la cámara recorra el pasillo suavemente.

----
:blue_book: **Funcion**
----

### **1. El Generador de Materiales (Lógica Condicional)**
Esta parte no solo crea colores, sino que toma "decisiones" basadas en el brillo:

<img width="490" height="51" alt="image" src="https://github.com/user-attachments/assets/79249ea5-a930-453b-939c-4b793f9a6937" />

- **¿Qué hace?** Analiza los canales R (Rojo) y B (Azul) del color que le envíes.

- **¿Por qué?** Si el color es intenso (como el azul eléctrico 0.0, 0.4, 1.0), el script decide automáticamente que ese material debe brillar. Esto ahorra tener que configurar manualmente la emisión para cada objeto decorativo.

### **2. La Trigonometría de la Curva**
Esta es la parte más compleja matemáticamente. Para colocar los cubos en un arco perfecto, el script usa el círculo unitario:

<img width="434" height="48" alt="image" src="https://github.com/user-attachments/assets/79f7fe1b-508d-4099-83e2-75cc1ebcafd3" />

- ``math.pi``: Representa 180°. El script calcula una fracción de ese ángulo para cada cubo (``j``).

- **Seno y Coseno:** Estas funciones convierten un ángulo en coordenadas **X** e **Y**.

- **Rotación:** ``rotacion_z = math.pi - angulo``. Esto es vital; sin esta línea, los cubos estarían todos "mirando" hacia el frente. Con esto, cada cubo rota para seguir la dirección de la curva.

### **3. El Sistema de Cámara "Follow Path"**
En lugar de animar la posición de la cámara cuadro por cuadro (lo cual sería una pesadilla en una curva), el script usa Constraints (Restricciones):

-** Cam_Path:**	Una curva invisible que sirve como "riel" o vía de tren.

- **Follow Path:**	El "pegamento" que une la cámara a la curva.

- **Offset Factor:**	El valor que va de 0.0 (inicio) a 1.0 (final). Es lo que realmente se anima.

----
:camera: **Capturas**
----

#### **Código**

<img width="585" height="454" alt="image" src="https://github.com/user-attachments/assets/985c69cb-f8b8-44f7-a609-63832a328b47" />

<img width="646" height="521" alt="image" src="https://github.com/user-attachments/assets/569b037a-089f-403d-b4c7-eed316c305c9" />

<img width="660" height="396" alt="image" src="https://github.com/user-attachments/assets/9703fae6-7819-42d4-a2ab-7e9ad5fb647f" />

<img width="465" height="582" alt="image" src="https://github.com/user-attachments/assets/e0381c19-645c-446f-af66-8303182fbcb2" />

<img width="647" height="305" alt="image" src="https://github.com/user-attachments/assets/29e37949-4b31-41ab-b8c4-a555c9560912" />

-------

#### **Resultado**
<img width="1182" height="504" alt="image" src="https://github.com/user-attachments/assets/9e98d119-ebd6-42cb-8865-bd5df6f6d8e9" />


