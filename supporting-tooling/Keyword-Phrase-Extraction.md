VBA code



Option Explicit
' =============================================================================
' Module: Keyword & Phrase Extraction with Anchor-Driven Expansion
' Author: Rishi (with Copilot assist)
' Purpose:
'   - Extract keywords, bigrams, trigrams from Sheet1!Column B.
'   - Exclude stopwords from keywords & phrases (for cleaner results).
'   - Skip numbers, times, dates, currency tokens, URLs/emails.
'   - Only output items with frequency >= MIN_FREQ (default: 3).
'   - NEW: Detect user-provided anchor bigrams/trigrams and expand them
'          by adding one word BEFORE or AFTER to form meaningful phrases.
'          (Optional: both-sides expansions via a flag.)
' Output:
'   - KeywordOutput sheet:
'       * A: Stop words (editable)
'       * C:D: Keywords
'       * F:G: Two-word phrases (bigrams)
'       * I:J: Three-word phrases (trigrams)
'       * K: Anchor Bigrams (editable)
'       * L: Anchor Trigrams (editable)
'       * N:O: Expanded phrases from Anchor Bigrams (3-word; +/- one word)
'       * P:Q: Expanded phrases from Anchor Trigrams (4-word; +/- one word)
' =============================================================================

Public Sub ExtractKeywordsAndPhrases_V4_Anchors()
    ' -----------------------------
    ' Configuration / Setup
    ' -----------------------------
    Dim wsSource As Worksheet, wsOutput As Worksheet
    Dim lastRow As Long, arr As Variant
    Dim r As Long

    ' Dictionaries for counting
    Dim keywordDict As Object        ' single tokens
    Dim bigramDict As Object         ' two-word phrases
    Dim trigramDict As Object        ' three-word phrases

    ' Anchor expansion dictionaries
    Dim expandFromBiDict As Object   ' expanded phrases derived from anchor bigrams
    Dim expandFromTriDict As Object  ' expanded phrases derived from anchor trigrams

    ' Stopword set (editable on output sheet)
    Dim stopSet As Object

    ' Anchor sets (editable on output sheet)
    Dim anchorBiSet As Object
    Dim anchorTriSet As Object

    ' Minimum frequency to display (>= 3 as requested)
    Dim MIN_FREQ As Long: MIN_FREQ = 3

    ' Behavior flags for anchor expansion
    Dim EXCLUDE_STOPWORD_CONTEXT As Boolean: EXCLUDE_STOPWORD_CONTEXT = True  ' exclude stopword in the added context word
    Dim INCLUDE_BOTH_SIDES As Boolean: INCLUDE_BOTH_SIDES = False             ' set True to also build X bigram Y and X trigram Y

    ' -----------------------------
    ' Source sheet & output sheet
    ' -----------------------------
    Set wsSource = ThisWorkbook.Sheets("Sheet1")

    ' Create or get output sheet named "KeywordOutput"
    On Error Resume Next
    Set wsOutput = ThisWorkbook.Sheets("KeywordOutput")
    On Error GoTo 0
    If wsOutput Is Nothing Then
        Set wsOutput = ThisWorkbook.Sheets.Add(After:=wsSource)
        wsOutput.Name = "KeywordOutput"
    End If
    
' Clear only output ranges, not anchor columns
wsOutput.Range("A:J").Clear
wsOutput.Range("N:Q").Clear


    ' -----------------------------
    ' Seed editable lists (Stopwords + Anchors)
    ' -----------------------------
    ' Stopwords (single-word tokens only). You can edit these cells and re-run.
    wsOutput.Range("A1").Value = "Stop Words (Edit Below)"
    wsOutput.Range("A2").Resize(39, 1).Value = Application.WorksheetFunction.Transpose(Array( _
        "a", "an", "and", "are", "as", "at", "be", "by", "for", "from", "has", "have", "he", "her", "his", "i", "in", "it", _
        "me", "my", "of", "on", "or", "our", "she", "that", "the", "their", "them", "there", "they", "this", "to", "we", "were", "with", "you"))

    ' Anchor bigrams & trigrams (editable). Put phrases exactly as they appear after cleaning (lowercase).
    wsOutput.Range("K1").Value = "Anchor Bigrams (Edit Below)"
    wsOutput.Range("L1").Value = "Anchor Trigrams (Edit Below)"
    ' These ranges are left blank for you to fill in (K2:K100, L2:L100)

    ' Build stopword set from the range (lowercased, trimmed)
    Set stopSet = LoadListToSet(wsOutput.Range("A2", wsOutput.Cells(wsOutput.Rows.Count, "A").End(xlUp)))
    ' Build anchor sets from ranges K and L
    Set anchorBiSet = LoadListToSet(wsOutput.Range("K2", wsOutput.Cells(wsOutput.Rows.Count, "K").End(xlUp)), True)
    Set anchorTriSet = LoadListToSet(wsOutput.Range("L2", wsOutput.Cells(wsOutput.Rows.Count, "L").End(xlUp)), True)

    ' -----------------------------
    ' Initialize dictionaries
    ' -----------------------------
    Set keywordDict = CreateObject("Scripting.Dictionary"): keywordDict.CompareMode = vbTextCompare
    Set bigramDict = CreateObject("Scripting.Dictionary"): bigramDict.CompareMode = vbTextCompare
    Set trigramDict = CreateObject("Scripting.Dictionary"): trigramDict.CompareMode = vbTextCompare
    Set expandFromBiDict = CreateObject("Scripting.Dictionary"): expandFromBiDict.CompareMode = vbTextCompare
    Set expandFromTriDict = CreateObject("Scripting.Dictionary"): expandFromTriDict.CompareMode = vbTextCompare

    ' -----------------------------
    ' Read Column B into an array for speed
    ' -----------------------------
    lastRow = wsSource.Cells(wsSource.Rows.Count, "B").End(xlUp).Row
    If lastRow < 2 Then
        MsgBox "No text found in Sheet1 Column B."
        Exit Sub
    End If
    arr = wsSource.Range("B2:B" & lastRow).Value

    ' -----------------------------
    ' Process each row: tokenize and count
    ' -----------------------------
    Dim text As String, tokens As Variant
    Dim i As Long, n As Long
    Dim w1 As String, w2 As String, w3 As String, wPrev As String, wNext As String

    For r = 1 To UBound(arr, 1)
        text = CStr(arr(r, 1))
        If Len(text) = 0 Then GoTo NextRow

        ' Clean and split into tokens
        tokens = CleanAndTokenize(text)
        n = UBound(tokens)
        If n < 0 Then GoTo NextRow

        ' Single-pass forward loop for keywords, bigrams, trigrams, and anchor expansions
        For i = 0 To n
            w1 = tokens(i)
            If IsSkippableToken(w1) Then GoTo NextI

            ' -------- Keywords: exclude stopwords --------
            If Not stopSet.Exists(w1) Then
                If keywordDict.Exists(w1) Then
                    keywordDict(w1) = keywordDict(w1) + 1
                Else
                    keywordDict(w1) = 1
                End If
            End If

            ' -------- Bigrams: w1 w2 --------
            If i < n Then
                w2 = tokens(i + 1)
                If Not IsSkippableToken(w2) Then
                    ' Exclude stopwords from general bigrams
                    If Not stopSet.Exists(w1) And Not stopSet.Exists(w2) Then
                        Dim bi As String: bi = w1 & " " & w2
                        If bigramDict.Exists(bi) Then
                            bigramDict(bi) = bigramDict(bi) + 1
                        Else
                            bigramDict.Add bi, 1
                        End If
                    End If

                    ' -------- Anchor Bigram Expansion ----------
                    ' If current bigram matches an anchor, build BEFORE/AFTER expansions
                    Dim anchorBi As String: anchorBi = w1 & " " & w2
                    If anchorBiSet.Exists(anchorBi) Then
                        ' BEFORE: wPrev + anchorBi  -> 3-word phrase
                        If i > 0 Then
                            wPrev = tokens(i - 1)
                            If Not IsSkippableToken(wPrev) Then
                                If (Not EXCLUDE_STOPWORD_CONTEXT) Or (Not stopSet.Exists(wPrev)) Then
                                    Dim expBeforeBi As String: expBeforeBi = wPrev & " " & anchorBi
                                    AddCount expandFromBiDict, expBeforeBi
                                End If
                            End If
                        End If
                        ' AFTER: anchorBi + wNext -> 3-word phrase
                        If i < n - 1 Then
                            wNext = tokens(i + 2)
                            If Not IsSkippableToken(wNext) Then
                                If (Not EXCLUDE_STOPWORD_CONTEXT) Or (Not stopSet.Exists(wNext)) Then
                                    Dim expAfterBi As String: expAfterBi = anchorBi & " " & wNext
                                    AddCount expandFromBiDict, expAfterBi
                                End If
                            End If
                        End If
                        ' BOTH-SIDES (optional): wPrev + anchorBi + wNext -> 4-word phrase
                        If INCLUDE_BOTH_SIDES And i > 0 And i < n - 1 Then
                            wPrev = tokens(i - 1)
                            wNext = tokens(i + 2)
                            If Not IsSkippableToken(wPrev) And Not IsSkippableToken(wNext) Then
                                If ((Not EXCLUDE_STOPWORD_CONTEXT) Or (Not stopSet.Exists(wPrev))) _
                                    And ((Not EXCLUDE_STOPWORD_CONTEXT) Or (Not stopSet.Exists(wNext))) Then
                                    Dim expBothBi As String: expBothBi = wPrev & " " & anchorBi & " " & wNext
                                    AddCount expandFromBiDict, expBothBi
                                End If
                            End If
                        End If
                    End If
                End If
            End If

            ' -------- Trigrams: w1 w2 w3 --------
            If i < n - 1 Then
                w2 = tokens(i + 1): w3 = tokens(i + 2)
                If Not IsSkippableToken(w2) And Not IsSkippableToken(w3) Then
                    ' Exclude stopwords from general trigrams
                    If Not stopSet.Exists(w1) And Not stopSet.Exists(w2) And Not stopSet.Exists(w3) Then
                        Dim tri As String: tri = w1 & " " & w2 & " " & w3
                        If trigramDict.Exists(tri) Then
                            trigramDict(tri) = trigramDict(tri) + 1
                        Else
                            trigramDict.Add tri, 1
                        End If
                    End If

                    ' -------- Anchor Trigram Expansion ----------
                    Dim anchorTri As String: anchorTri = w1 & " " & w2 & " " & w3
                    If anchorTriSet.Exists(anchorTri) Then
                        ' BEFORE: wPrev + anchorTri -> 4-word phrase
                        If i > 0 Then
                            wPrev = tokens(i - 1)
                            If Not IsSkippableToken(wPrev) Then
                                If (Not EXCLUDE_STOPWORD_CONTEXT) Or (Not stopSet.Exists(wPrev)) Then
                                    Dim expBeforeTri As String: expBeforeTri = wPrev & " " & anchorTri
                                    AddCount expandFromTriDict, expBeforeTri
                                End If
                            End If
                        End If
                        ' AFTER: anchorTri + wNext -> 4-word phrase
                        If i < n - 2 Then
                            wNext = tokens(i + 3)
                            If Not IsSkippableToken(wNext) Then
                                If (Not EXCLUDE_STOPWORD_CONTEXT) Or (Not stopSet.Exists(wNext)) Then
                                    Dim expAfterTri As String: expAfterTri = anchorTri & " " & wNext
                                    AddCount expandFromTriDict, expAfterTri
                                End If
                            End If
                        End If
                        ' BOTH-SIDES (optional): wPrev + anchorTri + wNext -> 5-word phrase
                        If INCLUDE_BOTH_SIDES And i > 0 And i < n - 2 Then
                            wPrev = tokens(i - 1)
                            wNext = tokens(i + 3)
                            If Not IsSkippableToken(wPrev) And Not IsSkippableToken(wNext) Then
                                If ((Not EXCLUDE_STOPWORD_CONTEXT) Or (Not stopSet.Exists(wPrev))) _
                                    And ((Not EXCLUDE_STOPWORD_CONTEXT) Or (Not stopSet.Exists(wNext))) Then
                                    Dim expBothTri As String: expBothTri = wPrev & " " & anchorTri & " " & wNext
                                    AddCount expandFromTriDict, expBothTri
                                End If
                            End If
                        End If
                    End If
                End If
            End If

NextI:
        Next i
NextRow:
    Next r

    ' -----------------------------
    ' Sort the dictionaries by frequency (descending)
    ' -----------------------------
    Dim sortedKeywords As Variant, sortedBigrams As Variant, sortedTrigrams As Variant
    Dim sortedExpBi As Variant, sortedExpTri As Variant

    sortedKeywords = SortDictionary(keywordDict)
    sortedBigrams = SortDictionary(bigramDict)
    sortedTrigrams = SortDictionary(trigramDict)
    sortedExpBi = SortDictionary(expandFromBiDict)
    sortedExpTri = SortDictionary(expandFromTriDict)

    ' -----------------------------
    ' Output results to KeywordOutput (only items >= MIN_FREQ)
    ' -----------------------------
    Dim outRow As Long, k As Long

    ' Keywords
    wsOutput.Range("C1:D1").Value = Array("Keyword", "Frequency")
    outRow = 2
    If Not IsEmpty(sortedKeywords) Then
        For k = LBound(sortedKeywords) To UBound(sortedKeywords)
            If CLng(sortedKeywords(k)(1)) >= MIN_FREQ Then
                wsOutput.Cells(outRow, 3).Value = sortedKeywords(k)(0)
                wsOutput.Cells(outRow, 4).Value = sortedKeywords(k)(1)
                outRow = outRow + 1
            End If
        Next k
    End If

    ' Bigrams
    wsOutput.Range("F1:G1").Value = Array("Two-Word Phrase", "Frequency")
    outRow = 2
    If Not IsEmpty(sortedBigrams) Then
        For k = LBound(sortedBigrams) To UBound(sortedBigrams)
            If CLng(sortedBigrams(k)(1)) >= MIN_FREQ Then
                wsOutput.Cells(outRow, 6).Value = sortedBigrams(k)(0)
                wsOutput.Cells(outRow, 7).Value = sortedBigrams(k)(1)
                outRow = outRow + 1
            End If
        Next k
    End If

    ' Trigrams
    wsOutput.Range("I1:J1").Value = Array("Three-Word Phrase", "Frequency")
    outRow = 2
    If Not IsEmpty(sortedTrigrams) Then
        For k = LBound(sortedTrigrams) To UBound(sortedTrigrams)
            If CLng(sortedTrigrams(k)(1)) >= MIN_FREQ Then
                wsOutput.Cells(outRow, 9).Value = sortedTrigrams(k)(0)
                wsOutput.Cells(outRow, 10).Value = sortedTrigrams(k)(1)
                outRow = outRow + 1
            End If
        Next k
    End If

    ' Expanded phrases from anchor bigrams (3-word, +/- one word)
    wsOutput.Range("N1:O1").Value = Array("Expanded (from Anchor Bigrams)", "Frequency")
    outRow = 2
    If Not IsEmpty(sortedExpBi) Then
        For k = LBound(sortedExpBi) To UBound(sortedExpBi)
            If CLng(sortedExpBi(k)(1)) >= MIN_FREQ Then
                wsOutput.Cells(outRow, 14).Value = sortedExpBi(k)(0)  ' Col N = 14
                wsOutput.Cells(outRow, 15).Value = sortedExpBi(k)(1)  ' Col O = 15
                outRow = outRow + 1
            End If
        Next k
    End If

    ' Expanded phrases from anchor trigrams (4-word, +/- one word)
    wsOutput.Range("P1:Q1").Value = Array("Expanded (from Anchor Trigrams)", "Frequency")
    outRow = 2
    If Not IsEmpty(sortedExpTri) Then
        For k = LBound(sortedExpTri) To UBound(sortedExpTri)
            If CLng(sortedExpTri(k)(1)) >= MIN_FREQ Then
                wsOutput.Cells(outRow, 16).Value = sortedExpTri(k)(0)  ' Col P = 16
                wsOutput.Cells(outRow, 17).Value = sortedExpTri(k)(1)  ' Col Q = 17
                outRow = outRow + 1
            End If
        Next k
    End If

    MsgBox "Extraction complete! Anchors expanded. Check 'KeywordOutput' (= " & MIN_FREQ & ")."
End Sub

' =============================================================================
' Helper: Add or increment a phrase count in a dictionary
' =============================================================================
Private Sub AddCount(ByRef d As Object, ByVal key As String)
    If d.Exists(key) Then
        d(key) = d(key) + 1
    Else
        d.Add key, 1
    End If
End Sub

' =============================================================================
' Helper: Load a list range into a Dictionary as a set (lowercase, trimmed)
' allowMultiword = True lets multi-word entries be used as-is (for anchors).
' =============================================================================
Private Function LoadListToSet(rng As Range, Optional allowMultiword As Boolean = False) As Object
    Dim d As Object, c As Range, s As String
    Set d = CreateObject("Scripting.Dictionary")
    d.CompareMode = vbTextCompare
    For Each c In rng
        s = Trim(LCase(CStr(c.Value)))
        If Len(s) > 0 Then
            If allowMultiword Then
                ' Keep full string (e.g., "customer lifetime value")
                d(s) = True
            Else
                ' For stopwords, ensure single tokens only (ignore spaces)
                If InStr(s, " ") = 0 Then d(s) = True
            End If
        End If
    Next c
    Set LoadListToSet = d
End Function

' =============================================================================
' Helper: Clean text and split into tokens
' - Removes common punctuation (keeps : / - as they can belong to dates/times)
' - Collapses multiple spaces
' - Lowercases results
' =============================================================================
Private Function CleanAndTokenize(ByVal text As String) As Variant
    Dim re As Object
    Set re = CreateObject("VBScript.RegExp")
    re.Global = True
    re.IgnoreCase = True

    ' Replace common punctuation with a space (retain colon/slash/hyphen)
    re.Pattern = "[\.,;!\?\(\)\[\]""'“”’]"
    text = re.Replace(text, " ")

    ' Collapse multiple spaces to single space
    re.Pattern = "\s+"
    text = LCase(Trim(re.Replace(text, " ")))

    CleanAndTokenize = Split(text, " ")
End Function

' =============================================================================
' Helper: Decide if a token is skippable
' - Numbers, percentages, times (e.g., 10:30, 9am), dates (e.g., 31/12/2025),
'   currency tokens, URLs/emails are skipped.
' =============================================================================
Private Function IsSkippableToken(ByVal w As String) As Boolean
    Dim re As Object
    Set re = CreateObject("VBScript.RegExp")
    re.IgnoreCase = True
    re.Global = False

    If Len(w) = 0 Then IsSkippableToken = True: Exit Function

    ' Pure numeric (allow digits with commas)
    re.Pattern = "^\d+[,\d]*$"
    If re.Test(w) Then IsSkippableToken = True: Exit Function

    ' Percentage (e.g., 12.5%)
    re.Pattern = "^\d+(\.\d+)?%$"
    If re.Test(w) Then IsSkippableToken = True: Exit Function

    ' Times (e.g., 9, 9am, 10:30, 10:30pm)
    re.Pattern = "^\d{1,2}(:\d{2})?\s*(am|pm)?$"
    If re.Test(w) Then IsSkippableToken = True: Exit Function

    ' Dates (e.g., 2025-12-31, 31/12/2025, 01-jan-25)
    re.Pattern = "^\d{1,4}[-/]\d{1,2}[-/]\d{1,4}$|^\d{1,2}[-/][a-z]{3}[-/]\d{2,4}$"
    If re.Test(w) Then IsSkippableToken = True: Exit Function

    ' Currency prefixes $, £, €, ?
    If Left$(w, 1) = "$" Or Left$(w, 1) = "£" Or Left$(w, 1) = "€" Or Left$(w, 1) = "?" Then
        IsSkippableToken = True: Exit Function
    End If

    ' URL or email tokens
    If InStr(1, w, "http", vbTextCompare) > 0 _
       Or InStr(1, w, "www.", vbTextCompare) > 0 _
       Or InStr(1, w, "@", vbBinaryCompare) > 0 Then
        IsSkippableToken = True: Exit Function
    End If
End Function

' =============================================================================
' Helper: Sort dictionary by frequency (descending)
' Returns an array of [key, value] pairs
' Handles empty dictionaries by returning Empty
' =============================================================================
Public Function SortDictionary(ByVal dict As Object) As Variant
    Dim cnt As Long: cnt = dict.Count
    If cnt = 0 Then
        SortDictionary = Empty
        Exit Function
    End If

    Dim keys As Variant, items As Variant
    Dim i As Long, j As Long
    Dim tempKey As Variant, tempVal As Variant
    Dim out() As Variant

    keys = dict.keys
    items = dict.items

    ' Bubble sort (sufficient for small dictionaries)
    For i = LBound(items) To UBound(items) - 1
        For j = i + 1 To UBound(items)
            If CLng(items(j)) > CLng(items(i)) Then
                tempVal = items(i): items(i) = items(j): items(j) = tempVal
                tempKey = keys(i): keys(i) = keys(j): keys(j) = tempKey
            End If
        Next j
    Next i

    ReDim out(LBound(keys) To UBound(keys))
    For i = LBound(keys) To UBound(keys)
        out(i) = Array(keys(i), items(i))
    Next i
    SortDictionary = out
End Function


