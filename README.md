# Electrode-calculator
For P-FM

package com.example.electrodecalculator

import android.content.Context
import android.graphics.Color
import android.os.Bundle
import android.text.Editable
import android.text.TextWatcher
import android.widget.*
import androidx.appcompat.app.AppCompatActivity
import net.objecthunter.exp4j.ExpressionBuilder

class MainActivity : AppCompatActivity() {

    private val variableIds = listOf(
        R.id.input_a, R.id.input_b, R.id.input_c, R.id.input_d,
        R.id.input_w, R.id.input_x, R.id.input_y, R.id.input_z
    )
    private val variableKeys = listOf("a", "b", "c", "d", "w", "x", "y", "z")

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val formulaEditText = findViewById<EditText>(R.id.formulaEditText)
        val formulaLabel = findViewById<TextView>(R.id.formulaLabel)
        val variableInputs = variableIds.map { findViewById<EditText>(it) }

        // 추가: 목표값 찾기 스위치와 입력란 바인딩
        val targetSwitch = findViewById<Switch>(R.id.targetSwitch)
        val targetValueEditText = findViewById<EditText>(R.id.targetValueEditText)

        // --- 변수 활성화/비활성화 함수 ---
        fun updateVariableInputEnabled() {
            val formula = formulaEditText.text.toString()
            // 수식에 실제로 사용된 변수만 추출
            val usedVars = variableKeys.filter { key ->
                Regex("""\b$key\b""").containsMatchIn(formula)
            }
            variableInputs.forEachIndexed { idx, editText ->
                if (usedVars.contains(variableKeys[idx])) {
                    editText.isEnabled = true
                    editText.setBackgroundColor(Color.WHITE)
                } else {
                    editText.isEnabled = false
                    editText.setBackgroundColor(Color.LTGRAY)
                    // setText("")를 하지 않음! (무한루프 방지)
                    // 만약 꼭 값을 지우고 싶으면 아래 주석 참고
                    /*
                    if (editText.text.toString() != "") {
                        editText.removeTextChangedListener(watcher)
                        editText.setText("")
                        editText.addTextChangedListener(watcher)
                    }
                    */
                }
            }
        }

        // 자동 계산 함수
        fun calculateAndShow() {
            updateVariableInputEnabled()

            val formula = formulaEditText.text.toString()
            val isTargetMode = targetSwitch.isChecked
            val targetValueText = targetValueEditText.text.toString()

            // 배경색 초기화
            variableInputs.forEach { it.setBackgroundColor(Color.TRANSPARENT) }

            if (formula.isBlank()) {
                formulaLabel.text = "F(x)="
                return
            }

            // 목표값 찾기 모드
            if (isTargetMode && targetValueText.isNotBlank()) {
                // 비어있는 변수 찾기
                val emptyIndices = variableInputs.mapIndexedNotNull { idx, et ->
                    if (et.isEnabled && et.text.toString().isBlank()) idx else null
                }
                if (emptyIndices.size == 1) {
                    val missingIdx = emptyIndices[0]
                    val missingKey = variableKeys[missingIdx]
                    val targetValue = targetValueText.toDoubleOrNull()
                    if (targetValue == null) {
                        formulaLabel.text = "F(x)= Target Value 오류"
                        return
                    }
                    try {
                        // 역산: 이분법(bisection) + 방향성 자동 판단
                        val exprBuilder = ExpressionBuilder(formula)
                        variableKeys.forEach { exprBuilder.variable(it) }
                        val expr = exprBuilder.build()
                        // 나머지 변수는 입력값으로 세팅
                        variableKeys.forEachIndexed { idx, key ->
                            if (idx != missingIdx) {
                                val value = variableInputs[idx].text.toString().toDoubleOrNull() ?: 0.0
                                expr.setVariable(key, value)
                            }
                        }
                        // 방향성 판단
                        val testLow = -1e3
                        val testHigh = 1e3
                        expr.setVariable(missingKey, testLow)
                        val fLow = expr.evaluate()
                        expr.setVariable(missingKey, testHigh)
                        val fHigh = expr.evaluate()
                        val fTarget = targetValue

                        // 해가 존재하는지 확인
                        val signLow = fLow - fTarget
                        val signHigh = fHigh - fTarget
                        if (signLow.isNaN() || signHigh.isNaN() || signLow * signHigh > 0) {
                            formulaLabel.text = "F(x)= 역산 실패"
                            return
                        }

                        // 이분법
                        var low = testLow
                        var high = testHigh
                        var answer = Double.NaN
                        val eps = 1e-7
                        var iter = 0
                        while (high - low > eps && iter < 100) {
                            val mid = (low + high) / 2
                            expr.setVariable(missingKey, mid)
                            val result = expr.evaluate()
                            if (result.isNaN()) break
                            if ((result - fTarget) * (fLow - fTarget) < 0) {
                                high = mid
                            } else {
                                low = mid
                            }
                            iter++
                        }
                        answer = (low + high) / 2

                        // 검증
                        expr.setVariable(missingKey, answer)
                        val check = expr.evaluate()
                        if (check.isNaN() || Math.abs(check - targetValue) > 1e-3 || answer.isNaN() || answer.isInfinite()) {
                            formulaLabel.text = "F(x)= 역산 실패"
                        } else {
                            // 결과 입력 및 강조
                            val answerStr = String.format("%.6f", answer)
                            variableInputs[missingIdx].setText(answerStr)
                            variableInputs[missingIdx].setBackgroundColor(Color.argb(180, 255, 255, 0)) // 노란색
                            formulaLabel.text = "F($missingKey)= $answerStr"
                        }
                    } catch (e: Exception) {
                        formulaLabel.text = "F(x)= 역산 오류"
                    }
                    return
                }
            }

            // 일반 계산 모드
            try {
                val exprBuilder = ExpressionBuilder(formula)
                variableKeys.forEach { exprBuilder.variable(it) }
                val expr = exprBuilder.build()
                variableKeys.forEachIndexed { idx, key ->
                    val value = variableInputs[idx].text.toString().toDoubleOrNull() ?: 0.0
                    expr.setVariable(key, value)
                }
                val result = expr.evaluate()
                formulaLabel.text = "F(x)= $result"
            } catch (e: Exception) {
                formulaLabel.text = "F(x)= 계산 오류"
            }
        }

        // 수식 입력란, 변수 입력란, 목표값 입력란에 변화가 있을 때마다 자동 계산
        val watcher = object : TextWatcher {
            override fun afterTextChanged(s: Editable?) { calculateAndShow() }
            override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {}
            override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {}
        }
        formulaEditText.addTextChangedListener(watcher)
        variableInputs.forEach { it.addTextChangedListener(watcher) }
        targetValueEditText.addTextChangedListener(watcher)
        targetSwitch.setOnCheckedChangeListener { _, _ -> calculateAndShow() }

        // 저장/불러오기 버튼 바인딩
        val saveButtons = listOf(
            findViewById<Button>(R.id.save1),
            findViewById<Button>(R.id.save2),
            findViewById<Button>(R.id.save3),
            findViewById<Button>(R.id.save4)
        )
        val loadButtons = listOf(
            findViewById<Button>(R.id.load1),
            findViewById<Button>(R.id.load2),
            findViewById<Button>(R.id.load3),
            findViewById<Button>(R.id.load4)
        )

        // 저장 버튼 클릭 리스너
        saveButtons.forEachIndexed { idx, button ->
            button.setOnClickListener {
                val formula = formulaEditText.text.toString()
                val variables = variableInputs.map { it.text.toString() }
                val prefs = getSharedPreferences("fx_prefs", Context.MODE_PRIVATE)
                val editor = prefs.edit()
                editor.putString("fx${idx+1}_formula", formula)
                variableKeys.forEachIndexed { vIdx, key ->
                    editor.putString("fx${idx+1}_$key", variables[vIdx])
                }
                editor.apply()
                Toast.makeText(this, "F(x) ${idx+1} 저장 완료!", Toast.LENGTH_SHORT).show()
            }
        }

        // 불러오기 버튼 클릭 리스너
        loadButtons.forEachIndexed { idx, button ->
            button.setOnClickListener {
                val prefs = getSharedPreferences("fx_prefs", Context.MODE_PRIVATE)
                val formula = prefs.getString("fx${idx+1}_formula", "") ?: ""
                val variables = variableKeys.map { key ->
                    prefs.getString("fx${idx+1}_$key", "") ?: ""
                }
                formulaEditText.setText(formula)
                variableInputs.forEachIndexed { vIdx, editText ->
                    editText.setText(variables[vIdx])
                }
                // 불러오면 자동 계산됨
                Toast.makeText(this, "F(x) ${idx+1} 불러오기 완료!", Toast.LENGTH_SHORT).show()
            }
        }
    }
}

