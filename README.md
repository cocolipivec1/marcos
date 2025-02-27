package com.example.calendarnotes

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.text.BasicTextField
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.lifecycle.viewmodel.compose.viewModel
import com.kizitonwose.calendar.compose.CalendarView
import com.kizitonwose.calendar.core.CalendarDay
import com.kizitonwose.calendar.core.firstDayOfWeekFromLocale
import kotlinx.coroutines.launch
import java.time.LocalDate
import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Entity
data class Note(
    @PrimaryKey val date: String,
    val content: String
)

@Dao
interface NoteDao {
    @Query("SELECT * FROM Note WHERE date = :date")
    fun getNoteByDate(date: String): Flow<Note?>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun ingenera
@Database(entities = [Note::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun noteDao(): NoteDao
}

class CalendarViewModel : ViewModel() {
    private val db = Room.databaseBuilder(
        MyApp.context,
        AppDatabase::class.java, "calendar-notes-db"
    ).build()

    private val noteDao = db.noteDao()
    var selectedDate by mutableStateOf(LocalDate.now())
    var noteText by mutableStateOf("")

    val noteFlow: Flow<Note?> = noteDao.getNoteByDate(selectedDate.toString())

    fun saveNote() {
        viewModelScope.launch {
            noteDao.insert(Note(selectedDate.toString(), noteText))
        }
    }
}

@Composable
fun CalendarScreen(viewModel: CalendarViewModel = viewModel()) {
    val coroutineScope = rememberCoroutineScope()
    val selectedDate = remember { mutableStateOf(LocalDate.now()) }
    val noteText = remember { mutableStateOf("") }
    val note by viewModel.noteFlow.collectAsState(initial = null)

    LaunchedEffect(viewModel.selectedDate) {
        noteText.value = note?.content ?: ""
    }

    Column(modifier = Modifier.padding(16.dp)) {
        CalendarView(
            startMonth = selectedDate.value.minusMonths(6),
            endMonth = selectedDate.value.plusMonths(6),
            firstDayOfWeek = firstDayOfWeekFromLocale()
        ) { day ->
            Button(onClick = { viewModel.selectedDate = day.date }) {
                Text(day.date.dayOfMonth.toString())
            }
        }
        Spacer(modifier = Modifier.height(16.dp))
        BasicTextField(
            value = noteText.value,
            onValueChange = { noteText.value = it },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(modifier = Modifier.height(8.dp))
        Button(onClick = {
            viewModel.noteText = noteText.value
            coroutineScope.launch { viewModel.saveNote() }
        }) {
            Text("Guardar Nota")
        }
    }
}

class MyApp : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { CalendarScreen() }
    }
}

@Preview
@Composable
fun PreviewCalendarScreen() {
    CalendarScreen()
}
