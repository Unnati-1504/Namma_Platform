# Namma_Platform
Namma-Platform: A Kannada-first railway assistant for rural passengers. Features include a high-contrast coach layout and Kannada audio announcements.
package com.example.namma_platform






import android.graphics.Color
import android.os.Bundle
import android.speech.tts.TextToSpeech
import android.view.View
import android.widget.*
import androidx.appcompat.app.AppCompatActivity
import androidx.core.graphics.toColorInt // This fixes the KTX warning
import org.json.JSONArray
import java.io.InputStream
import java.util.Locale

class MainActivity : AppCompatActivity(), TextToSpeech.OnInitListener {

    private lateinit var tts: TextToSpeech
    private lateinit var stationSpinner: Spinner
    private lateinit var tvTrainName: TextView
    private lateinit var tvPlatformInfo: TextView
    private lateinit var coachContainer: LinearLayout
    private lateinit var btnHelpMe: Button

    private var trainDataList = mutableListOf<Map<String, Any>>()
    private var selectedTrainIndex = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        stationSpinner = findViewById(R.id.stationSpinner)
        tvTrainName = findViewById(R.id.tvTrainName)
        tvPlatformInfo = findViewById(R.id.tvPlatformInfo)
        coachContainer = findViewById(R.id.coachContainer)
        btnHelpMe = findViewById(R.id.btnHelpMe)

        tts = TextToSpeech(this, this)

        loadJsonData()
        setupSpinner()

        btnHelpMe.setOnClickListener {
            speakAnnouncement()
        }
    }

    private fun loadJsonData() {
        try {
            val inputStream: InputStream = assets.open("trains.json")
            val jsonString = inputStream.bufferedReader().use { it.readText() }
            val jsonArray = JSONArray(jsonString)

            for (i in 0 until jsonArray.length()) {
                val obj = jsonArray.getJSONObject(i)
                val coachesJson = obj.getJSONArray("coaches")
                val coachesList = mutableListOf<String>()
                for (j in 0 until coachesJson.length()) {
                    coachesList.add(coachesJson.getString(j))
                }

                val map = mapOf(
                    "station" to obj.getString("station"),
                    "nextTrain" to obj.getString("nextTrain"),
                    "platform" to obj.getString("platform"),
                    "coaches" to coachesList
                )
                trainDataList.add(map)
            }
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

    private fun setupSpinner() {
        val stations = trainDataList.map { it["station"].toString() }
        val adapter = ArrayAdapter(this, android.R.layout.simple_spinner_item, stations)
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        stationSpinner.adapter = adapter

        stationSpinner.onItemSelectedListener = object : AdapterView.OnItemSelectedListener {
            override fun onItemSelected(p0: AdapterView<*>?, p1: View?, position: Int, p3: Long) {
                selectedTrainIndex = position
                updateUI(position)
            }
            override fun onNothingSelected(p0: AdapterView<*>?) {}
        }
    }

    private fun updateUI(index: Int) {
        val currentTrain = trainDataList[index]
        
        // Success Criteria: Platform and Train Name matching train name
        tvTrainName.text = "Train: ${currentTrain["nextTrain"]}"
        tvPlatformInfo.text = "Platform: ${currentTrain["platform"]}"

        coachContainer.removeAllViews()
        val coaches = currentTrain["coaches"] as List<*>

        for (coach in coaches) {
            val coachName = coach.toString()
            val tv = TextView(this)
            tv.text = coachName
            tv.textSize = 18f
            tv.setPadding(30, 20, 30, 20)
            
            val params = LinearLayout.LayoutParams(
                LinearLayout.LayoutParams.WRAP_CONTENT,
                LinearLayout.LayoutParams.WRAP_CONTENT
            )
            params.setMargins(10, 0, 10, 0)
            tv.layoutParams = params

            // Fixed: Using String.toColorInt() KTX extension
            when (coachName) {
                "Engine" -> {
                    tv.setBackgroundColor(Color.RED)
                    tv.setTextColor(Color.WHITE)
                }
                "Ladies" -> {
                    tv.setBackgroundColor(Color.MAGENTA)
                    tv.setTextColor(Color.WHITE)
                }
                else -> {
                    // Success Criteria: High contrast Blue/Yellow UI
                    tv.setBackgroundColor("#FFDC00".toColorInt()) 
                    tv.setTextColor("#001F3F".toColorInt())
                }
            }
            coachContainer.addView(tv)
        }
    }

    override fun onInit(status: Int) {
        if (status == TextToSpeech.SUCCESS) {
            // Success Criteria: Kannada audio
            val result = tts.setLanguage(Locale("kn", "IN"))
            if (result == TextToSpeech.LANG_MISSING_DATA || result == TextToSpeech.LANG_NOT_SUPPORTED) {
                tts.language = Locale.US
            }
        }
    }

    private fun speakAnnouncement() {
        val currentTrain = trainDataList[selectedTrainIndex]
        val announcementText = "ಗಮನಿಸಿ. ಮುಂದಿನ ರೈಲು ${currentTrain["nextTrain"]}, ಪ್ಲಾಟ್‌ಫಾರ್ಮ್ ಸಂಖ್ಯೆ ${currentTrain["platform"]} ಗೆ ಬರಲಿದೆ."
        tts.speak(announcementText, TextToSpeech.QUEUE_FLUSH, null, null)
    }

    override fun onDestroy() {
        if (::tts.isInitialized) {
            tts.stop()
            tts.shutdown()
        }
        super.onDestroy()
    }
}
