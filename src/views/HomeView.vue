<template>
  <div class="container" dir="rtl">
    <div class="brand">
      <h1><span>Ahmed</span> Ismail</h1>
    </div>
    <div class="brand">
      </div>

    <div v-if="!isRadarActive" id="setupScreen">
      <div class="box">
        <div class="box-title">📍 نقطة الرصد</div>
        <button class="btn-gps" @click="fetchGPS" id="gpsBtn">
          {{ gpsBtnText }}
        </button>
        
        <div v-if="myLat" id="myCoords" class="coords-box">
          Lat: {{ myLat.toFixed(6) }}<br>
          Lon: {{ myLon.toFixed(6) }}
        </div>
        
        <button v-if="myLat" class="btn-copy" @click="copyCoords">
          {{ copyBtnText }}
        </button>
      </div>

      <div class="box">
        <div class="box-title">🎯 الهدف</div>
        <textarea 
          v-model="targetInput" 
          rows="2" 
          placeholder="الصق الإحداثيات هنا (مثال: 31.2, 29.9)"
        ></textarea>
      </div>
      
      <button class="btn-start" @click="initRadar">📡 تشغيل الرادار</button>
    </div>

    <div v-if="isRadarActive" id="radarScreen" class="radar-ui">
      <div class="compass-ring" :class="{ 'locked': isLocked }" id="ring">
        <div class="fixed-pointer"></div>
        
        <div 
          class="target-arrow-container" 
          :style="{ transform: `rotate(${relativeTargetAngle}deg)` }"
        >
          <div class="target-arrow"></div>
        </div>

        <div class="compass-labels" :style="{ transform: `rotate(${-smoothedHeading}deg)` }">
          <div class="compass-mark mark-n">N</div>
          <div class="compass-mark mark-s">S</div>
          <div class="compass-mark mark-e">E</div>
          <div class="compass-mark mark-w">W</div>
        </div>

        <div class="val-big">{{ Math.round(currentHeading) }}°</div>
        <div class="val-small">{{ cardinalDirection }}</div>
      </div>

      <div class="data-row">
        <div class="data-column">
          <div class="label">زاوية الهدف</div>
          <div class="data-val">{{ Math.round(targetAzimuth) }}°</div>
        </div>
        <div class="data-column border-right">
          <div class="label">المسافة</div>
          <div class="data-val" style="color: #3498db;">{{ distanceText }}</div>
        </div>
      </div>

      <button class="btn-gps" style="margin-top:20px;" @click="resetRadar">🔄 إعادة ضبط</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // إحداثيات المستخدم
      myLat: null,
      myLon: null,
      // إحداثيات الهدف
      targetInput: "",
      tarLat: null,
      tarLon: null,
      // حالة الرادار
      isRadarActive: false,
      gpsBtnText: "🧭 تحديد موقعي الحالي",
      copyBtnText: "📋 نسخ الإحداثيات",
      // الحسابات
      targetAzimuth: 0,
      smoothedHeading: 0,
      currentHeading: 0,
      isLocked: false,
      relativeTargetAngle: 0,
      distance: 0,
      wasLocked: false
    };
  },
  computed: {
    cardinalDirection() {
      const h = (this.currentHeading + 360) % 360;
      if (h >= 337.5 || h < 22.5) return "شمال";
      if (h >= 22.5 && h < 67.5) return "شمال شرقي";
      if (h >= 67.5 && h < 112.5) return "شرق";
      if (h >= 112.5 && h < 157.5) return "جنوب شرقي";
      if (h >= 157.5 && h < 202.5) return "جنوب";
      if (h >= 202.5 && h < 247.5) return "جنوب غربي";
      if (h >= 247.5 && h < 292.5) return "غرب";
      if (h >= 292.5 && h < 337.5) return "شمال غربي";
      return "";
    },
    distanceText() {
      return this.distance > 0 ? `${Math.round(this.distance)} م` : "--- م";
    }
  },
  methods: {
    fetchGPS() {
      this.gpsBtnText = "⏳ جاري التحديد...";
      navigator.geolocation.watchPosition(
        (pos) => {
          this.myLat = pos.coords.latitude;
          this.myLon = pos.coords.longitude;
          this.gpsBtnText = "🔄 تحديث الموقع";
        },
        (err) => { 
          console.error(err); // استخدمنا err هنا
          alert("تأكد من تفعيل الموقع أو السماح بالوصول");
        },
        { enableHighAccuracy: true }
      );
    },
    copyCoords() {
      if (!this.myLat) return;
      const msg = `📍 إحداثيات نقطة الرصد:\nLat: ${this.myLat.toFixed(6)}\nLon: ${this.myLon.toFixed(6)}\n---\nSurveyApp by Ahmed Ismail`;
      navigator.clipboard.writeText(msg).then(() => {
        this.copyBtnText = "✔️ تم النسخ";
        setTimeout(() => { this.copyBtnText = "📋 نسخ الإحداثيات"; }, 2000);
      });
    },
    async initRadar() {
      if (!this.myLat) return alert("حدد موقعك أولاً!");
      
      const coords = this.targetInput.match(/-?\d+\.\d+/g);
      if (!coords || coords.length < 2) return alert("إحداثيات الهدف غير صحيحة");
      
      this.tarLat = parseFloat(coords[0]);
      this.tarLon = parseFloat(coords[1]);

      if (typeof DeviceOrientationEvent !== 'undefined' && typeof DeviceOrientationEvent.requestPermission === 'function') {
        const permission = await DeviceOrientationEvent.requestPermission();
        if (permission === 'granted') this.startCompass();
      } else {
        this.startCompass();
      }

      this.calculateTarget();
      this.isRadarActive = true;
    },
    calculateTarget() {
      const toRad = (d) => (d * Math.PI) / 180;
      const dLon = toRad(this.tarLon - this.myLon);
      
      // حساب الزاوية (Bearing)
      const y = Math.sin(dLon) * Math.cos(toRad(this.tarLat));
      const x = Math.cos(toRad(this.myLat)) * Math.sin(toRad(this.tarLat)) -
                Math.sin(toRad(this.myLat)) * Math.cos(toRad(this.tarLat)) * Math.cos(dLon);
      
      this.targetAzimuth = (Math.atan2(y, x) * 180 / Math.PI + 360) % 360;

      // حساب المسافة (Haversine)
      const R = 6371000;
      const dLat = toRad(this.tarLat - this.myLat);
      const a = Math.sin(dLat / 2) ** 2 +
                Math.cos(toRad(this.myLat)) * Math.cos(toRad(this.tarLat)) *
                Math.sin(toRad(this.tarLon - this.myLon) / 2) ** 2;
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
      this.distance = R * c;
    },
    startCompass() {
      const handler = (e) => {
        let heading = e.webkitCompassHeading || (360 - e.alpha);
        if (heading === null) return;

        // الفلتر السلس
        let diff = heading - this.smoothedHeading;
        if (diff < -180) diff += 360;
        if (diff > 180) diff -= 360;
        this.smoothedHeading += diff * 0.05;

        this.currentHeading = (this.smoothedHeading + 360) % 360;

        // حساب زاوية سهم الهدف بالنسبة لاتجاه الموبايل
        this.relativeTargetAngle = (this.targetAzimuth - this.smoothedHeading + 360) % 360;

        // التنبيه عند التطابق (Lock)
        let diffTarget = Math.abs((this.targetAzimuth - this.currentHeading + 540) % 360 - 180);
        if (diffTarget <= 3) {
          this.isLocked = true;
          if (navigator.vibrate && !this.wasLocked) {
            navigator.vibrate(25);
            this.wasLocked = true;
          }
        } else {
          this.isLocked = false;
          this.wasLocked = false;
        }
      };
      window.addEventListener("deviceorientationabsolute", handler, true);
      window.addEventListener("deviceorientation", handler, true);
    },
    resetRadar() {
      this.isRadarActive = false;
      this.targetInput = "";
      this.targetAzimuth = 0;
      this.isLocked = false;
      this.wasLocked = false;
    }
  }
};
</script>

<style scoped>
/* المتغيرات الأساسية للألوان */
:export {
  --primary-blue: #00d2ff;
  --dark-bg: #0a0e14;
  --card-bg: #161c24;
  --text-gray: #919eab;
  --success-green: #00ab55;
}

.container {
  min-height: 100vh;
  background-color: #0a0e14;
  color: #fff;
  font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
 
}

/* التنسيق الخاص بالصناديق */
.box {
  background: #161c24;
  border-radius: 16px;
  padding: 20px;
  margin-bottom: 20px;
  width: 100%;
  max-width: 400px;
  border: 1px solid #2d3748;
  box-shadow: 0 8px 16px rgba(0,0,0,0.4);
}

.box-title {
  font-size: 0.9rem;
  color: #919eab;
  margin-bottom: 15px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 1px;
}

/* الأزرار */
button {
  width: 100%;
  padding: 12px;
  border-radius: 12px;
  border: none;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
  margin-top: 10px;
  font-size: 1rem;
}

.btn-gps {
  background: rgba(0, 210, 255, 0.1);
  color: #00d2ff;
  border: 1px solid rgba(0, 210, 255, 0.3);
}

.btn-gps:hover {
  background: rgba(0, 210, 255, 0.2);
}

.btn-start {
  background: linear-gradient(135deg, #00d2ff 0%, #3a7bd5 100%);
  color: white;
  box-shadow: 0 4px 14px rgba(0, 210, 255, 0.4);
}

.btn-start:active {
  transform: scale(0.98);
}

/* الرادار - الجزء الأهم */
.radar-ui {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

.compass-ring {
  width: 280px;
  height: 280px;
  border-radius: 50%;
  border: 4px solid #1a222c;
  position: relative;
  margin: 30px 0;
  background: radial-gradient(circle, #1a222c 0%, #0a0e14 100%);
  box-shadow: 0 0 40px rgba(0, 210, 255, 0.15);
  display: flex;
  justify-content: center;
  align-items: center;
}

/* تأثير الـ Lock عند تطابق الزاوية */
.compass-ring.locked {
  border-color: #00ab55;
  box-shadow: 0 0 50px rgba(0, 171, 85, 0.3);
}

.fixed-pointer {
  position: absolute;
  top: -15px;
  width: 0;
  height: 0;
  border-left: 10px solid transparent;
  border-right: 10px solid transparent;
  border-bottom: 20px solid #ff4842;
  z-index: 10;
}

.compass-labels {
  position: absolute;
  width: 100%;
  height: 100%;
  transition: transform 0.1s linear;
}

.compass-mark {
  position: absolute;
  font-weight: 900;
  color: #fff;
  font-size: 1.2rem;
}

.mark-n { top: 10px; left: 50%; transform: translateX(-50%); color: #ff4842; }
.mark-s { bottom: 10px; left: 50%; transform: translateX(-50%); }
.mark-e { right: 10px; top: 50%; transform: translateY(-50%); }
.mark-w { left: 10px; top: 50%; transform: translateY(-50%); }

.target-arrow-container {
  position: absolute;
  width: 100%;
  height: 100%;
  transition: transform 0.1s linear;
  z-index: 5;
}

.target-arrow {
  position: absolute;
  top: 30px;
  left: 50%;
  transform: translateX(-50%);
  width: 4px;
  height: 40px;
  background: #00ab55;
  border-radius: 2px;
  box-shadow: 0 0 15px #00ab55;
}

.target-arrow::before {
  content: '';
  position: absolute;
  top: -10px;
  left: -6px;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-bottom: 12px solid #00ab55;
}

/* القيم الرقمية */
.val-big {
  font-size: 3.5rem;
  font-weight: 800;
  font-family: 'Courier New', monospace;
  color: #00d2ff;
}

.val-small {
  position: absolute;
  bottom: 60px;
  font-size: 1rem;
  color: #919eab;
}

/* صف البيانات السفلي */
.data-row {
  display: flex;
  background: #161c24;
  border-radius: 16px;
  width: 100%;
  max-width: 400px;
  padding: 15px 0;
  border: 1px solid #2d3748;
}

.data-column {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.label {
  font-size: 0.75rem;
  color: #919eab;
}

.data-val {
  font-size: 1.4rem;
  font-weight: bold;
  font-family: 'Courier New', monospace;
}

/* الحقول النصية */
textarea {
  width: 100%;
  background: #0a0e14;
  border: 1px solid #2d3748;
  border-radius: 8px;
  color: #fff;
  padding: 10px;
  font-family: monospace;
  resize: none;
}

textarea:focus {
  outline: none;
  border-color: #00d2ff;
}
#setupScreen{
     display: flex;
     justify-content: center;
     align-items: center;
     border: 2px solid  #00d2ff;
     border-radius: 10px;
     gap: 20px;
     flex-direction: column;
     width: 500px;
     height: 500px;
}
#setupScreen {
  display: flex;
  justify-content: center;
  align-items: center;
  border: 2px solid #00d2ff;
  border-radius: 20px; /* خليها أنعم شوية */
  gap: 20px;
  flex-direction: column;
  width: 90%; /* بياخد 90% من عرض الشاشة في الموبايل */
  max-width: 500px; /* وميزدش عن 500 في الشاشات الكبيرة */
  padding: 30px 15px; /* مساحة داخلية عشان العناصر متنلزقش في الحواف */
  margin: auto;
}
.container {
  min-height: 100vh;
  background-color: #0a0e14;
  color: #fff;
  padding: 10px; /* قللنا البادينج شوية للموبايل */
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  box-sizing: border-box; /* مهم جداً عشان الحسابات تطلع صح */
}
@media (max-width: 360px) {
  .compass-ring {
    width: 240px; /* تصغير حجم الرادار في الشاشات الضيقة جداً */
    height: 240px;
  }
  .val-big {
    font-size: 2.5rem; /* تصغير الخط عشان ميتكسرش */
  }
}
.box, .data-row {
  width: 95%; 
  max-width: 400px;
  box-sizing: border-box; /* عشان الـ padding ميطلعش العنصر بره الشاشة */
}
/* تنسيق قسم البراند */
.brand {
  margin-top: 20px;
  margin-bottom: -10px;
  text-align: center;
}

h1 {
  font-family: 'Orbitron', 'Segoe UI', sans-serif; /* لو تقدر تستخدم خط Orbitron من جوجل هيبقى تحفة */
  font-size: 2.5rem;
  font-weight: 800;
  color: #fff;
  letter-spacing: 3px;
  text-transform: uppercase;
  margin: 0;
  /* تأثير الظل المضيء */
  text-shadow: 0 0 10px rgba(0, 210, 255, 0.5),
               0 0 20px rgba(0, 210, 255, 0.2);
}

h1 span {
  color: #00d2ff; /* اللون اللي اخترناه للرادار */
  position: relative;
}

/* حركة بسيطة (Animation) تخلي الاسم كأنه شاشة بتنور وتطفي */
h1 span::after {
  content: '';
  position: absolute;
  bottom: -5px;
  left: 0;
  width: 100%;
  height: 2px;
  background: #00d2ff;
  box-shadow: 0 0 10px #00d2ff;
  animation: scan 2s infinite alternate;
}

@keyframes scan {
  from { opacity: 0.2; width: 0%; }
  to { opacity: 1; width: 100%; }
}

/* الميديا كويري عشان الاسم ميكسرش في الموبايل */
@media (max-width: 480px) {
  h1 {
    font-size: 1.8rem;
    letter-spacing: 1px;
  }
}
</style>