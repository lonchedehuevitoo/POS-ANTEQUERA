<template>
  <div class="caja-view">
    <h1 class="titulo">Caja de Cobro</h1>
    

    <section class="productos">
      <h2>Productos disponibles</h2>
      <div v-for="producto in productos" :key="producto.id" class="producto-card">
        <div class="info-producto">
          <img
            v-if="producto.imagen"
            :src="producto.imagen"
            alt="Imagen producto"
            class="imagen-producto"
          />
          <div>
            <h3>{{ producto.nombre }}</h3>
            <p class="precio">$ {{ producto.precio.toFixed(2) }}</p>
            <p class="stock" v-if="producto.stock !== undefined">
              Stock: {{ producto.stock }}
            </p>
            <p class="codigo">Código: {{ producto.codigo }}</p>
          </div>
        </div>
        <button @click="agregarAlCarrito(producto)" class="btn-agregar" :disabled="producto.stock === 0">
          Agregar
        </button>
      </div>
    </section>

    <!-- Input para escanear código de barras -->
<div class="lector-codigo">
  <label for="codigoBarra">Escanear Código de Barras:</label>
  <input
    id="codigoBarra"
    v-model="codigoEscaneado"
    type="text"
    inputmode="numeric"
    pattern="\d*"
    maxlength="13"
    @input="filtrarNumeros"
    @keyup.enter="procesarEscaneo"
    placeholder="Escanea o escribe el código y presiona Enter"
    ref="inputCodigo"
    @blur="reenfocarInput"
    autofocus
  />
</div>


    <section class="carrito">
      <h2>Carrito</h2>
      <div v-if="carrito.length === 0" class="empty-carrito">No hay productos en el carrito.</div>

      <ul v-else class="lista-carrito">
        <li v-for="item in carrito" :key="item.id" class="item-carrito">
          <span>{{ item.nombre }}</span>
          <div class="contador-cantidad">
            <button @click="cambiarCantidad(item, -1)" :disabled="item.cantidad === 1">-</button>
            <span>{{ item.cantidad }}</span>
            <button @click="cambiarCantidad(item, 1)" :disabled="item.cantidad >= item.stock">+</button>
          </div>
          <span class="subtotal">$ {{ (item.precio * item.cantidad).toFixed(2) }}</span>
        </li>
      </ul>

      <p class="total">Total: ${{ total.toFixed(2) }}</p>

      <div class="metodo-pago">
        <label>Método de pago:</label>
        <label>
          <input type="radio" value="efectivo" v-model="metodoPago" />
          Efectivo
        </label>
        <label>
          <input type="radio" value="tarjeta" v-model="metodoPago" />
          Tarjeta
        </label>
      </div>

      <div v-if="totalAcumulado > 0" class="recuadro-total-acumulado">
  <p><strong>Total acumulado del día:</strong> ${{ totalAcumulado.toFixed(2) }}</p>
</div>

      <button
        @click="finalizarVenta"
        :disabled="carrito.length === 0"
        class="btn-finalizar"
      >
        Finalizar Venta
      </button>
    </section>

    <section v-if="ventaFinalizada" class="ticket" id="ticket">
      <h2>Ticket de Venta</h2>
      <ul>
        <li v-for="item in carritoAnterior" :key="item.id">
          {{ item.nombre }} (x{{ item.cantidad }}) - ${{ (item.precio * item.cantidad).toFixed(2) }}
        </li>
      </ul>
      <p><strong>Total: ${{ totalAnterior.toFixed(2) }}</strong></p>
      <p>Método de pago: {{ metodoPagoAnterior }}</p>
      <p>¡Gracias por su compra!</p>

      <button @click="imprimirTicket" class="btn-imprimir">Imprimir Ticket</button>
    </section>
  </div>
</template>

<script>
export default {
  data() {
    return {
      productos: [],
      carrito: [],
      codigoEscaneado: '',
      metodoPago: 'efectivo',
      ventaFinalizada: false,
      carritoAnterior: [],
      metodoPagoAnterior: '',
      totalAnterior: 0,
      totalAcumulado: 0,
      cierresCaja: [],
      ventasRealizadas: []  // Agregado para controlar ventas del día
    };
  },
  computed: {
    total() {
      return this.carrito.reduce((suma, item) => suma + item.precio * item.cantidad, 0);
    }
  },
  mounted() {
    this.fetchInventario();

    const fechaGuardada = localStorage.getItem('fecha_total_acumulado');
    const totalGuardado = localStorage.getItem('total_acumulado_dia');
    const cierresGuardados = localStorage.getItem('cierres_caja');

    const hoy = new Date().toISOString().slice(0, 10);

    if (fechaGuardada !== hoy) {
      // Día nuevo, reiniciar acumulados y cargar cierres previos
      this.totalAcumulado = 0;
      this.ventasRealizadas = [];
      if (cierresGuardados) {
        this.cierresCaja = JSON.parse(cierresGuardados);
      }
      localStorage.setItem('fecha_total_acumulado', hoy);
      localStorage.setItem('total_acumulado_dia', '0');
    } else {
      // Mismo día, cargar datos guardados
      this.totalAcumulado = totalGuardado ? parseFloat(totalGuardado) : 0;
      this.ventasRealizadas = JSON.parse(localStorage.getItem('ventas_realizadas')) || [];
      this.cierresCaja = cierresGuardados ? JSON.parse(cierresGuardados) : [];
    }

    this.$nextTick(() => {
      this.$refs.inputCodigo?.focus();
    });
    window.addEventListener('keydown', this.detectarEscape);
  },

  beforeUnmount() {
    window.removeEventListener('keydown', this.detectarEscape);
  },
  methods: {
    async fetchInventario() {
      try {
        const response = await fetch('/api/productos.json');
        if (!response.ok) throw new Error('Error al obtener productos');
        this.productos = await response.json();
      } catch (error) {
        console.error('Fallo al obtener inventario:', error);
        this.productos = [
          {
            id: 1,
            nombre: 'AGUA',
            precio: 25,
            stock: 20,
            codigo: '758104005796',
            imagen: 'https://superfarmaciasbali.com/cdn/shop/files/Bonafont1L.jpg?v=1717198870'
          },
        ];
      }
    },
    agregarAlCarrito(producto) {
      if (producto.stock === 0) {
        alert('Producto sin stock');
        return;
      }
      const encontrado = this.carrito.find(p => p.id === producto.id);
      if (encontrado) {
        if (encontrado.cantidad < producto.stock) {
          encontrado.cantidad++;
        } else {
          alert('No hay más stock disponible');
        }
      } else {
        this.carrito.push({ ...producto, cantidad: 1 });
      }
      this.ventaFinalizada = false;
    },
    cambiarCantidad(item, delta) {
      const nuevaCantidad = item.cantidad + delta;
      if (nuevaCantidad < 1) return;

      const original = this.productos.find(p => p.id === item.id);
      if (original && nuevaCantidad > original.stock) {
        alert('No hay más stock disponible');
        return;
      }

      item.cantidad = nuevaCantidad;
    },
    finalizarVenta() {
      // Actualizar stock de productos
      this.carrito.forEach(item => {
        const prod = this.productos.find(p => p.id === item.id);
        if (prod) prod.stock -= item.cantidad;
      });

      const venta = {
        id: Date.now(),
        productos: JSON.parse(JSON.stringify(this.carrito)),
        total: this.total,
        metodoPago: this.metodoPago,
        fecha: new Date().toISOString()
      };

      // Guardar venta en arreglo y localStorage
      this.ventasRealizadas.push(venta);
      localStorage.setItem('ventas_realizadas', JSON.stringify(this.ventasRealizadas));

      // Acumular total y guardar
      this.totalAcumulado += this.total;
      localStorage.setItem('total_acumulado_dia', this.totalAcumulado.toString());

      // Guardar fecha para reiniciar luego
      const hoy = new Date().toISOString().slice(0, 10);
      localStorage.setItem('fecha_total_acumulado', hoy);

      // Emitir evento para App.vue
      this.$emit('nueva-venta', venta);

      // Guardar datos para mostrar el ticket
      this.carritoAnterior = [...this.carrito];
      this.totalAnterior = this.total;
      this.metodoPagoAnterior = this.metodoPago;
      this.ventaFinalizada = true;

      // Vaciar carrito y limpiar código
      this.carrito = [];
      this.codigoEscaneado = '';
    },
cerrarCaja() {
  const cierre = {
    fecha: new Date().toISOString().slice(0, 10),
    totalVentas: this.totalAcumulado,
    ventas: [...this.ventasRealizadas],
    cajero: localStorage.getItem('usuario_logueado') || 'Sin nombre'
  };

  this.cierresCaja.push(cierre);
  localStorage.setItem('cierres_caja', JSON.stringify(this.cierresCaja));

  this.totalAcumulado = 0;
  this.ventasRealizadas = [];
  localStorage.setItem('ventas_realizadas', '[]');
  localStorage.setItem('total_acumulado_dia', '0');
  localStorage.setItem('fecha_total_acumulado', cierre.fecha);
},

    imprimirTicket() {
      const original = document.body.innerHTML;
      const ticket = document.getElementById('ticket').innerHTML;
      document.body.innerHTML = ticket;
      window.print();
      document.body.innerHTML = original;
      window.location.reload();
    },
    procesarEscaneo() {
      const codigo = this.codigoEscaneado.trim();
      if (!codigo) return;

      const producto = this.productos.find(p => p.codigo === codigo);

      if (!producto) {
        alert('Producto no encontrado');
        this.codigoEscaneado = '';
        return;
      }

      this.agregarAlCarrito(producto);
      this.codigoEscaneado = '';
    },
    reenfocarInput() {
      setTimeout(() => {
        this.$refs.inputCodigo?.focus();
      }, 100);
    },
    detectarEscape(event) {
      if (event.key === 'Escape') {
        this.reenfocarInput();
      }
    }
  }
}

</script>


<style scoped>

.caja-view h1 {
  text-align: center;
  font-family: 'Poppins', 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  font-weight: 700;          /* Negrita elegante */
  font-size: 2.4rem;         /* Más grande y legible */
  letter-spacing: 0.5px;     /* Ligero espaciado */
  color: #2c3e50;            /* Mantén tu color corporativo */
  margin-bottom: 1.2rem;     /* Separación con los controles */
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.05); /* Sombra sutil */
}

.recuadro-total-acumulado {
  margin-top: 1rem;
  background-color: #f1f8e9;
  border: 2px solid #aed581;
  border-radius: 10px;
  padding: 1rem;
  color: #33691e;
  font-size: 1.2rem;
  text-align: center;
  font-weight: bold;
}


.caja-view {
  max-width: 900px;
  margin: auto;
  padding: 1.5rem;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 0 15px rgba(0,0,0,0.1);
}

.lector-codigo {
  margin-bottom: 1rem;
  text-align: center;
}

.lector-codigo input {
  padding: 0.5rem 1rem;
  font-size: 1.1rem;
  width: 300px;
  max-width: 90vw;
  border-radius: 6px;
  border: 2px solid #4caf50;
  outline: none;
  transition: border-color 0.3s ease;
}
.lector-codigo input:focus {
  border-color: #2e7d32;
}

.productos, .carrito, .ticket {
  background: white;
  border-radius: 8px;
  padding: 1rem 1.5rem;
  margin-bottom: 2rem;
  box-shadow: 0 1px 6px rgba(0,0,0,0.12);
}

.producto-card {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.8rem 0;
  border-bottom: 1px solid #eee;
}

.info-producto {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.imagen-producto {
  width: 70px;
  height: 70px;
  object-fit: cover;
  border-radius: 6px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.15);
}

.precio {
  font-weight: 600;
  color: #4caf50;
}

.stock, .codigo {
  font-size: 0.85rem;
  color: #666;
}

.btn-agregar {
  background-color: #4caf50;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}
.btn-agregar:disabled {
  background-color: #a5d6a7;
  cursor: not-allowed;
}
.btn-agregar:hover:not(:disabled) {
  background-color: #388e3c;
}

.carrito ul {
  list-style: none;
  padding-left: 0;
  margin-top: 0.5rem;
}

.item-carrito {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 0;
  border-bottom: 1px solid #eee;
}

.contador-cantidad {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.contador-cantidad button {
  width: 26px;
  height: 26px;
  background-color: #4caf50;
  border: none;
  border-radius: 3px;
  color: white;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.contador-cantidad button:disabled {
  background-color: #9e9e9e;
  cursor: not-allowed;
}

.contador-cantidad button:hover:not(:disabled) {
  background-color: #388e3c;
}

.subtotal {
  min-width: 70px;
  text-align: right;
  font-weight: 600;
  color: #333;
}

.total {
  font-size: 1.3rem;
  font-weight: 700;
  text-align: right;
  margin-top: 1rem;
  color: #2e7d32;
}

.metodo-pago {
  margin-top: 1rem;
  display: flex;
  gap: 1.5rem;
  align-items: center;
}

.metodo-pago label {
  cursor: pointer;
  user-select: none;
}

.btn-finalizar {
  margin-top: 1.5rem;
  width: 100%;
  background-color: #2e7d32;
  color: white;
  font-weight: 700;
  padding: 0.8rem 0;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}
.btn-finalizar:disabled {
  background-color: #a5d6a7;
  cursor: not-allowed;
}
.btn-finalizar:hover:not(:disabled) {
  background-color: #1b5e20;
}

.ticket h2 {
  color: #2e7d32;
}

.btn-imprimir {
  margin-top: 1rem;
  background-color: #4caf50;
  color: white;
  border: none;
  padding: 0.6rem 1rem;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}
.btn-imprimir:hover {
  background-color: #388e3c;
}

/* Estilos impresión */
@media print {
  body * {
    visibility: hidden;
  }
  #ticket, #ticket * {
    visibility: visible;
  }
  #ticket {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
  }
}
</style>