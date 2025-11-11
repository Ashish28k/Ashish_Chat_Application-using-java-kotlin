package com.example.chatapp;

import android.os.Bundle;
import android.widget.*;
import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import com.example.chatapp.adapters.MessageAdapter;
import com.example.chatapp.models.MessageModel;
import com.example.chatapp.models.UserModel;
import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.database.*;
import com.squareup.picasso.Picasso;

import java.util.*;

public class ChatActivity extends AppCompatActivity {

    private EditText etMessage;
    private Button btnSend;
    private ListView messageListView;
    private ArrayList<MessageModel> messages;
    private MessageAdapter adapter;

    private String receiverUid, senderUid;
    private DatabaseReference messagesRef, usersRef;

    private ImageView profileImage;
    private TextView userName, userStatus;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_chat);

        receiverUid = getIntent().getStringExtra("receiverUid");
        senderUid = FirebaseAuth.getInstance().getUid();
        messages = new ArrayList<>();

        // Initialize Views
        etMessage = findViewById(R.id.etMessage);

        ImageButton btnSend = findViewById(R.id.btnSend);

        messageListView = findViewById(R.id.messageListView);
        profileImage = findViewById(R.id.profileImage);
        userName = findViewById(R.id.tvName);
        userStatus = findViewById(R.id.tvStatus);

        adapter = new MessageAdapter(this, messages, senderUid);
        messageListView.setAdapter(adapter);

        messagesRef = FirebaseDatabase.getInstance().getReference("Messages");
        usersRef = FirebaseDatabase.getInstance().getReference("Users");

        String chatPath = senderUid + "_" + receiverUid;

        // Fetch User Info (Receiver's Profile)
        usersRef.child(receiverUid).addListenerForSingleValueEvent(new ValueEventListener() {
            @Override
            public void onDataChange(@NonNull DataSnapshot snapshot) {
                UserModel receiver = snapshot.getValue(UserModel.class);
                if (receiver != null) {
                    userName.setText(receiver.getName());
                    userStatus.setText(receiver.getStatus());
                    // Load profile image using Picasso (you can use any other image loading library)
                    if (!receiver.getImageUrl().equals("default")) {
                        Picasso.get().load(receiver.getImageUrl()).into(profileImage);
                    }
                }
            }

            @Override
            public void onCancelled(@NonNull DatabaseError error) {
                // Handle errors (optional)
            }
        });

        // Listen for new messages
        messagesRef.child(chatPath).addValueEventListener(new ValueEventListener() {
            @Override
            public void onDataChange(@NonNull DataSnapshot snapshot) {
                messages.clear();
                for (DataSnapshot ds : snapshot.getChildren()) {
                    MessageModel msg = ds.getValue(MessageModel.class);
                    messages.add(msg);
                }
                adapter.notifyDataSetChanged();
                messageListView.setSelection(messages.size() - 1); // Scroll to the bottom
            }

            @Override
            public void onCancelled(@NonNull DatabaseError error) {
                // Handle errors
            }
        });

        // Handle message send action
        btnSend.setOnClickListener(v -> {
            String text = etMessage.getText().toString().trim();
            if (!text.isEmpty()) {
                long timestamp = System.currentTimeMillis();
                MessageModel msg = new MessageModel(senderUid, receiverUid, text, timestamp);

                String key = messagesRef.child(chatPath).push().getKey();
                if (key != null) {
                    messagesRef.child(chatPath).child(key).setValue(msg);

                    // Mirror the message for the receiver
                    String reversePath = receiverUid + "_" + senderUid;
                    messagesRef.child(reversePath).child(key).setValue(msg);

                    etMessage.setText("");  // Clear input field after sending
                    messageListView.setSelection(messages.size() - 1); // Scroll to the bottom
                }
            }
        });
    }
}
