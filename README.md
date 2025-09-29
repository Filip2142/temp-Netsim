# temp-Netsim


#include <fstream>
#include <iostream>
#include <string>
#include <deque>

-----------------------------------------------
// Adaptivní threshold management
double m_adaptLastBufferLevel = 0;
int m_adaptNegativeTrendCount = 0;
double m_adaptThrZone = 2.4;     // pásmo, kdy začít zvedat threshold (např. 110 % thr)    2.7 //1.4
static constexpr int m_adaptTrendRequired = 1;     // kolik cyklů musí být záporný trend po sobě              1  ///1
EventId m_adaptThresholdEvent;
uint32_t m_initialThreshold = 0;
uint32_t m_adaptPositiveTrendCount = 0;

enum class AdaptMode { RISING, STABLE };
AdaptMode m_adaptMode = AdaptMode::STABLE;
double m_emaBufferDelta = 0.0;

bool m_isConsumingKeys = false;
uint32_t m_noConsumeCounter = 0;

// Status — pokud už máš definováno, použij stávající
// Status m_currentStatus;
--------------------------------------
///////////////////// pridano
    .AddTraceSource ("Mmax_printChange",
                 "Trace s horní hranicí pro threshold (growGuard/Mmax) pro vykreslení",
                 MakeTraceSourceAccessor (&QKDBuffer::m_mmaxPrintChangeTrace),
                 "ns3::QKDBuffer::Mmax_printChange")
---------------------------------------------------

m_thresholdKeyBit = Mthr; ///////////////////////////////
--------------------------------------------------------

SetMthr(Mthr); ///

    ////////////////////////////KEV//////////////////////////
    // Spuštění adaptivní logiky po inicializaci bufferu:
    m_adaptLastBufferLevel = static_cast<double>(m_currentKeyBit);
    m_adaptNegativeTrendCount = 0;
    m_initialThreshold = m_thresholdKeyBit;
    std::cout << "zacina se volat AdaptiveThresholdUpdate" << std::endl;
    // Když poprvé nastavuješ threshold:
    m_originalThresholdKeyBit = m_thresholdKeyBit;

    m_adaptThresholdEvent = Simulator::Schedule(Seconds(1.0), &QKDBuffer::AdaptiveThresholdUpdate, this);

}

void QKDBuffer::PeriodicStatsUpdate(double now) {
    //double now = Simulator::Now().GetSeconds();

    // Průběžné ukládání hodnot můžeš dělat stále zde...
    // appendBufferValue(m_currentValue);

    // Pokud je čas větší nebo roven 69s, vypiš statistiky a už nevypisuj znovu
    static bool statsPrinted = false;
    if (!statsPrinted && now >= 49.0) {
        //PrintBufferStatistics("buffer_stats.txt");
        statsPrinted = true;
    }
    std::cout << "VOLA SE PeriodicStatsUpdate/////////////////////////" << std::endl;
}


void QKDBuffer::PrintBufferStatistics(std::string filename, double now)
{
    /*
    // Vyhodnotíme minimum, maximum a průměr z historických hodnot
    uint32_t minBuffer = UINT32_MAX;
    uint32_t maxBuffer = 0;
    double sumBuffer = 0.0;
    size_t count = m_previousValues.size();

    for (const auto& val : m_previousValues) {
        if (val.value < minBuffer) minBuffer = val.value;
        if (val.value > maxBuffer) maxBuffer = val.value;
        sumBuffer += val.value;
    }
    double avgBuffer = (count > 0) ? sumBuffer / count : 0.0;

    std::cout << "----- QKDBuffer Statistics -----" << std::endl;
    std::cout << "Min buffer level: " << minBuffer << " bits" << std::endl;
    std::cout << "Max buffer level: " << maxBuffer << " bits" << std::endl;
    std::cout << "Avg buffer level: " << avgBuffer << " bits" << std::endl;
    std::cout << "--------------------------------" << std::endl;
    */
    // Statické proměnné pro globální minimum a průměr
    static uint64_t sumBufferAllTime = 0;
    static uint32_t countBufferAllTime = 0;
    static uint32_t minBufferAllTime = UINT32_MAX;

    uint32_t currentBuffer = m_currentKeyBit;

    sumBufferAllTime += currentBuffer;
    countBufferAllTime += 1;
    if (currentBuffer < minBufferAllTime && m_isConsumingKeys)
        minBufferAllTime = currentBuffer;

    double avgBufferAllTime = (countBufferAllTime > 0) ?
        static_cast<double>(sumBufferAllTime) / countBufferAllTime : 0.0;

    std::cout << "------------------------------" << std::endl;
    std::cout << "Min buffer level (global): " << minBufferAllTime << " bits" << std::endl;
    std::cout << "Avg buffer level (global): " << avgBufferAllTime << " bits" << std::endl;
    std::cout << "------------------------------" << std::endl;


    if (now >= 48.0)
    {
        // Výpis do konzole nebo do souboru
    std::ostream* out;
    std::ofstream file;
    if (!filename.empty()) {
        file.open(filename, std::ios::app);
        out = &file;
    } else {
        out = &std::cout;
    }

    (*out) << "----- QKDBuffer Statistics -----" << std::endl;
    (*out) << "Min buffer level: " << minBufferAllTime << " bits" << std::endl;
    (*out) << "Avg buffer level: " << avgBufferAllTime << " bits" << std::endl;
    (*out) << "--------------------------------" << std::endl;

        if (file.is_open())
            file.close();
    }
    else
    {
        Simulator::Schedule(Seconds(1.0),
        &QKDBuffer::PrintBufferStatistics,
        this,
        filename,
        Simulator::Now().GetSeconds());
    }

-----------------------------------------------------------------
//pridano manag 13.8.
bool QKDBuffer::CanAppReleaseKey(double appReleaseInterval) const
{
    
    const double currentTime = Simulator::Now().GetSeconds();
    const double thr   = static_cast<double>(m_thresholdKeyBit);
    const double level = static_cast<double>(m_currentKeyBit);

    // Odběr pouze nad threshold+rezerva, pouze READY/CHARGING (interval záměrně vypnutý)
    if (level >= thr * (1.1 + m_appMinBufferOverThr)
        && (m_status == QKDSTATUS_READY || m_status == QKDSTATUS_CHARGING))
    {
        return true;
    }
    return false;
    //return true; /// pokud chci neblokovat aplikaci, tak vracim pouze TRUE"""
}

//pridano manag
void QKDBuffer::NotifyAppKeyRelease()
{
    const double now = Simulator::Now().GetSeconds();
    m_lastAppReleaseTime = now;

    // GRACE impuls pro detekci spotřeby – pomůže zachytit bursty
    m_lastConsumeTs   = now;
    m_isConsumingKeys = true;
}

---------------------------------------------------
//////////////////funkcni

void  //14.8. aktualni a funkcni
QKDBuffer::AdaptiveThresholdUpdate ()
{
    
    double currentBufferLevel = static_cast<double>(m_currentKeyBit);
    double threshold = static_cast<double>(m_thresholdKeyBit);
    double maxBuffer = static_cast<double>(m_maxKeyBit);
    double delta = currentBufferLevel - m_adaptLastBufferLevel;

    std::cout << "--------------------[DEBUG_INIT] currentKeyBit=" << m_currentKeyBit
          << " thresholdKeyBit=" << m_thresholdKeyBit
          << " maxKeyBit=" << m_maxKeyBit
          << std::endl;


    double zoneMin = 1.1; // spodni hranice zony
    double zoneMax = m_adaptThrZone; //horni hranice


    ///////////////////////doplneno 14.8.


    // --- PEVNÝ STROP PRO THRESHOLD (jednoduchý a dostačující) ---
    // headroom nad startovním prahem a minimem + cap na část kapacity
    const double kStartHeadroom = 0.20; // +50 % nad počátečním thr
    const double kMinHeadroom   = 0.50; // +100 % nad minimem (tj. 2× minimum)
    const double kCapFrac       = 0.70; // nikdy nad 90 % kapacity
    const double kHystFrac      = 0.02; // 2 % hystereze proti pilce u stropu

    const double startThr = static_cast<double>(m_originalThresholdKeyBit);
    // pokud máš ve třídě m_minKeyBit, použij ho; jinak nech minFromStart = startThr
    #ifdef HAVE_QKDBUFFER_MINKEYBIT
    const double minThr   = static_cast<double>(m_minKeyBit);
    #else
    const double minThr   = startThr; // fallback
    #endif

    const double baseFromStart = startThr * (1.0 + kStartHeadroom);
    const double baseFromMin   = minThr   * (1.0 + kMinHeadroom);
    const double capLimit      = maxBuffer * kCapFrac;

    const double Mmax = std::min(std::max(baseFromStart, baseFromMin), capLimit);
    Mmax_print = Mmax;
    //const double Mmax = 65000;
    const double growGuard = Mmax * (1.0 - kHystFrac); // nad to už nerůst

    ///////////trace"""
    m_mmaxPrintChangeTrace(static_cast<uint32_t>(Mmax)); // horní hranice pro THR

    std::cout << "/////////////Mmax = " << Mmax << " ,currentKeyBit = " << m_currentKeyBit << std::endl;

    // když by byl historicky nad stropem, okamžitě sraz na Mmax
    if (threshold > Mmax) {
        uint32_t clampThr = static_cast<uint32_t>(Mmax);
        if (clampThr != m_thresholdKeyBit) {
            std::cout << "AdaptiveThreshold: clamp to Mmax=" << clampThr
                      << " (buffer=" << currentBufferLevel << ")\n";
            SetMthr(clampThr);
            threshold = static_cast<double>(clampThr);
        }
    }
    ///////////////////////////////////////////////////////

    double simTime = Simulator::Now().GetSeconds();

    // V threshold update:
    double increaseFactor = 1.05; // default pro BASELINE

    ///////////////////test
    m_adaptThrZone = 2.4;
    //////////////////////

    // --- LOG pro ladění ---
    std::cout << "DEBUG: time=" << simTime 
              << " isConsuming=" << m_isConsumingKeys 
              << " buffer=" << currentBufferLevel 
              << " threshold=" << threshold 
              << " originalThr=" << m_originalThresholdKeyBit 
              << " delta=" << delta 
              << " status=" << m_status << std::endl;


    std::cout << "[ADAPT] t=" << simTime
          << " buffer=" << currentBufferLevel
          << " threshold=" << threshold
          << " adaptZone=" << threshold * m_adaptThrZone
          << " delta=" << delta
          << " status=" << m_status
          << " trendCount=" << m_adaptNegativeTrendCount
          << std::endl;

    if (m_adaptThrZone == 1.4) 
    {         // Aggressive
        increaseFactor = 1.18;           // větší skok (18 %)
    } 
    else if (m_adaptThrZone == 2.0) 
    {  // Baseline
        increaseFactor = 1.10;           // střední skok (10 %)
    } 
    else if (m_adaptThrZone == 2.4) 
    {  // Conservative
        increaseFactor = 1.05;           // menší skok (5 %)
    }


    // --- Stabilizace: počáteční okno 5s ---
    if (simTime < 5.0) {
        m_adaptLastBufferLevel = currentBufferLevel;
        Simulator::Schedule(Seconds(1.0), &QKDBuffer::AdaptiveThresholdUpdate, this);
        return;
    }

    // --- Zvýšení threshold ---
    if ((m_status == QKDBuffer::QKDSTATUS_READY || m_status == QKDBuffer::QKDSTATUS_WARNING)
        && currentBufferLevel <= threshold * zoneMax
        && currentBufferLevel >= threshold * zoneMin
        && delta < 0
        && m_isConsumingKeys
        && threshold < growGuard) // nerůst, pokud jsme u stropu
    {
        m_adaptNegativeTrendCount++;

        if (m_adaptNegativeTrendCount >= m_adaptTrendRequired)  
        {
            //uint32_t newThr = static_cast<uint32_t>(threshold * increaseFactor); // +5 % (threshold * 1.10)
            //if (newThr > static_cast<uint32_t>(maxBuffer * 0.40)) {
            //    newThr = static_cast<uint32_t>(maxBuffer * 0.40);
            //}

            /////////////////////////////////////
             // [CHANGE] sjednocený limit: clampni kandidáta na growGuard (zrušeno 0.40/0.95 cap)
            //double cand = threshold * increaseFactor;
            //if (cand > growGuard) cand = growGuard;                   // jediný strop
            //uint32_t newThr = static_cast<uint32_t>(cand);

            //////////////////////////////////


        
        // ✅ [CHANGE] tvrdý strop: nikdy nepřes Mmax (zabrání jednorázovému přestřelu)
        //if (newThr > static_cast<uint32_t>(Mmax)) {
        //    newThr = static_cast<uint32_t>(Mmax);
        //}
         
        //if(newThr != m_thresholdKeyBit) {
        //        std::cout << "AdaptiveThreshold: Increasing threshold to " << newThr 
        //                  << " (buffer=" << currentBufferLevel << ")" << std::endl;
        //        SetMthr(newThr);
        //        //threshold = cand;    // [CHANGE] lokální update
        //    }

        // [FIX] Zachovej původní násobek, ale VŽDY ořízni na Mmax
        double cand = threshold * increaseFactor;               // původní politika
        if (cand > Mmax) cand = Mmax;                           // <<< klíčový clamp
        uint32_t newThr = static_cast<uint32_t>(cand);

        // [CLEANUP] odstraň starý cap na 0.40/0.95*maxBuffer – je teď nadbytečný
        // if (newThr > static_cast<uint32_t>(maxBuffer * 0.40)) newThr = ...;

        if (newThr != m_thresholdKeyBit) {
            std::cout << "AdaptiveThreshold: Increasing threshold to " << newThr
                      << " (buffer=" << currentBufferLevel
                      << ", Mmax=" << Mmax
                      << ", growGuard=" << growGuard << ")\n";
            SetMthr(newThr);
            // (volitelně) threshold = cand;  // pouze lokální proměnná pro další výpočty
        }
        

            m_adaptNegativeTrendCount = 0;
        }
    }
    else {
        m_adaptNegativeTrendCount = 0;
    }

    // --- EWMA relaxace zpět ---
    if ((delta >= 0.0) && (m_thresholdKeyBit > m_originalThresholdKeyBit))
    {
        double alpha = 0.2;  // jemnější relaxace
        double targetThr = static_cast<double>(m_originalThresholdKeyBit);
        double relaxedThr = alpha * targetThr + (1.0 - alpha) * threshold;

        
        //uint32_t newThr = static_cast<uint32_t>(relaxedThr);
        
        //if (newThr < m_originalThresholdKeyBit) {
        //    newThr = m_originalThresholdKeyBit;
        //}

        //if (newThr != m_thresholdKeyBit) {
        //    std::cout << "AdaptiveThreshold: EWMA relax to " << newThr 
        //              << " (buffer=" << currentBufferLevel << ")" << std::endl;
        //    SetMthr(newThr);
        //}

        // [CHANGE] clamp i při relaxaci na growGuard (strop jednotně platí všude)
        if (relaxedThr > growGuard) relaxedThr = growGuard;

        uint32_t newThr = static_cast<uint32_t>(relaxedThr);

        if (newThr < m_originalThresholdKeyBit) {
            newThr = m_originalThresholdKeyBit;
        }

        if (newThr != m_thresholdKeyBit) {
            std::cout << "AdaptiveThreshold: EWMA relax to " << newThr 
                      << " (buffer=" << currentBufferLevel << ")" << std::endl;
            SetMthr(newThr);
            threshold = relaxedThr;                                    // [CHANGE] lokální update
        }
        

    }

    std::cout<<"CurrentBufferLevel: " << currentBufferLevel << std::endl;
    // --- Update trendu ---
    m_adaptLastBufferLevel = currentBufferLevel;

    //pro appku
    currentBufferLevelApp = currentBufferLevel;
    //currentBufferThrApp = newThr;


    // --- Update příznaku spotřeby ---
    if (m_isConsumingKeys) {
        m_noConsumeCounter = 0;
    } else {
        m_noConsumeCounter++;
        if (m_noConsumeCounter >= 5) {
            m_isConsumingKeys = false;
        }
    }

    // --- Periodické volání ---
    m_adaptThresholdEvent = Simulator::Schedule(Seconds(1.0), &QKDBuffer::AdaptiveThresholdUpdate, this);
    
}
------------------------------------------------
///////////////////////////////////pridano/////////////
void
QKDBuffer::SetConsumingKeys (bool state)
{
    m_isConsumingKeys = state;
}




Ptr<QKDKey>
QKDBuffer::FetchKeyBySize (const uint32_t& keySize)
{
    NS_LOG_FUNCTION(this << keySize << m_currentKeyBit << m_currentKeyBitReally);
    NS_LOG_INFO("FETCH CALLED: keySize=" << keySize << " currentBuffer=" << m_currentKeyBit);
    std::cout <<  "FETCH CALLED: keySize=" << keySize << " currentBuffer=" << m_currentKeyBit << std::endl;

    Ptr<QKDKey> key = 0;

    for (std::map<std::string, Ptr<QKDKey> >::iterator it = m_keys.begin(); it != m_keys.end(); ++it)
    {
        if (it->second->GetState() == QKDKey::READY && it->second->GetSizeInBits() == keySize)
        {
            key = it->second;

            // Odebereme klíč z bufferu
            m_keys.erase(it);

            // --- DŮLEŽITÉ! Aktualizujeme stav bufferu ---
            m_currentKeyBit -= key->GetSizeInBits();

            //--aktivita APP----
            m_kevActivityCounter++;
            m_isConsumingKeys = true;


            // Indikace spotřeby pouze pokud buffer je nad threshold (tedy reálný provoz)
            /*
            if (m_currentKeyBit > m_thresholdKeyBit) {
                m_isConsumingKeys = true;
            }*/

            // Statistiky
            UpdateKeyConsumptionStatistics(key);

            // Přepočítání případných derivací
            KeyCalculation();

            break;
        }
    }

    if (!key)
    {
        NS_FATAL_ERROR(this << " Key of desired length is not available in the QKD buffer!");
    }

    return key;
}

--------------------------------------------------------
void 
QKDBuffer::CheckState(void)
{
    NS_LOG_FUNCTION  (this << m_minKeyBit << m_currentKeyBit << m_currentKeyBitPrevious << m_thresholdKeyBit << m_maxKeyBit << m_status << m_previousStatus );

    double deltaBuffer = static_cast<double>(m_currentKeyBit) - static_cast<double>(m_currentKeyBitPrevious);

    if (m_currentKeyBit >= m_thresholdKeyBit) { 
        if (deltaBuffer > 0 && m_isRisingCurve == true) {
            NS_LOG_FUNCTION  ("case 1A - CHARGING");
            m_status = QKDBuffer::QKDSTATUS_CHARGING;
        } else {
            NS_LOG_FUNCTION  ("case 1B - READY");
            m_status = QKDBuffer::QKDSTATUS_READY;

            ///---doplneno reset isConsumingKeys pri vstupu do READY ---
            if (m_previousStatus != QKDBuffer::QKDSTATUS_READY) {
                m_isConsumingKeys = false;
                m_noConsumeCounter = 0;
                NS_LOG_INFO("DEBUG: Transition to READY → Reset isConsumingKeys to false");
            }

        }

    } else if (m_currentKeyBit < m_thresholdKeyBit && m_currentKeyBit > m_minKeyBit && 
        ((m_isRisingCurve == true && m_previousStatus != QKDBuffer::QKDSTATUS_READY) || m_previousStatus == QKDBuffer::QKDSTATUS_EMPTY )
    ) {
        NS_LOG_FUNCTION  ("case 2 - CHARGING");
        m_status = QKDBuffer::QKDSTATUS_CHARGING;

    } else if (m_currentKeyBit < m_thresholdKeyBit && m_currentKeyBit > m_minKeyBit && 
        (m_previousStatus != QKDBuffer::QKDSTATUS_CHARGING)
    ) { 
        NS_LOG_FUNCTION  ("case 3 - WARNING");
        m_status = QKDBuffer::QKDSTATUS_WARNING;

    } else if (m_currentKeyBit <= m_minKeyBit) { 
        NS_LOG_FUNCTION  ("case 4 - EMPTY");
        m_status = QKDBuffer::QKDSTATUS_EMPTY;

    } else { 
        NS_LOG_FUNCTION  ("case UNDEFINED"     << m_minKeyBit << m_currentKeyBit << m_currentKeyBitPrevious << m_thresholdKeyBit << m_maxKeyBit << m_status << m_previousStatus ); 
    } 

    if (m_previousStatus != m_status) {
        NS_LOG_FUNCTION  (this << "STATUS IS NOT EQUAL TO PREVIOUS STATUS" << m_previousStatus << m_status);
        NS_LOG_FUNCTION  (this << m_minKeyBit << m_currentKeyBit << m_currentKeyBitPrevious << m_thresholdKeyBit << m_maxKeyBit << m_status << m_previousStatus );
 
        m_StatusChangeTrace(m_previousStatus);
        m_StatusChangeTrace(m_status);
        m_previousStatus = m_status;
    } 
}
--------------------------------------------
///pro appku
    currentBufferThrApp = thr;
-----------------------







